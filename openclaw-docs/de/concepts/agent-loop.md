---
read_when:
    - Sie benötigen eine genaue schrittweise Beschreibung der Agentenschleife oder der Lebenszyklusereignisse.
    - Sie ändern die Sitzungswarteschlange, das Schreiben von Transkripten oder das Verhalten der Schreibsperre für Sitzungen
summary: Lebenszyklus des Agenten-Loops, Streams und Wartesemantik
title: Agentenschleife
x-i18n:
    generated_at: "2026-07-26T18:19:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1d0102ffb6ebf572ea0201470db138775be33b0f0b655d9d08742177be5f3f31
    source_path: concepts/agent-loop.md
    workflow: 16
---

Der Agenten-Loop ist der serialisierte, sitzungsbezogene Lauf, der eine Nachricht in
Aktionen und eine Antwort umwandelt: Annahme, Kontextzusammenstellung, Modellinferenz, Werkzeugausführung,
Streaming, Persistierung.

## Einstiegspunkte

- Gateway-RPC: `agent` und `agent.wait`.
- CLI: `openclaw agent`.

## Ablauf eines Laufs

1. `agent`-RPC validiert Parameter, löst die Sitzung auf (`sessionKey`/`sessionId`), persistiert Sitzungsmetadaten und gibt sofort `{ runId, acceptedAt }` zurück.
2. `agentCommand` führt den Turn aus: löst die Standardwerte für Modell, Denken, Ausführlichkeit und Ablaufverfolgung auf, lädt den Skills-Snapshot, ruft `runEmbeddedAgent` auf und gibt ersatzweise **Lebenszyklusende/-fehler** aus, falls der eingebettete Loop dies nicht bereits getan hat.
3. `runEmbeddedAgent`: Serialisiert Läufe über sitzungsbezogene und globale Warteschlangen, löst Modell und Authentifizierungsprofil auf, erstellt die OpenClaw-Sitzung, abonniert Laufzeitereignisse, streamt Assistenten-/Werkzeug-Deltas, erzwingt das Laufzeitlimit (mit Abbruch nach dessen Ablauf) und gibt Nutzdaten sowie Nutzungsmetadaten zurück. Bei Turns des Codex-App-Servers bricht es außerdem einen angenommenen Turn ab, der vor einem terminalen Ereignis keinen weiteren App-Server-Fortschritt mehr erzeugt.
4. `subscribeEmbeddedAgentSession` überführt Laufzeitereignisse in den `agent`-Stream: Werkzeugereignisse in `stream: "tool"`, Assistenten-Deltas in `stream: "assistant"`, Lebenszyklusereignisse in `stream: "lifecycle"` (`phase: "start" | "end" | "error"`).
5. `agent.wait` (`waitForAgentRun`) wartet auf **Lebenszyklusende/-fehler** für eine `runId` und gibt `{ status: ok|error|timeout, startedAt, endedAt, error? }` zurück.

## Warteschlangen und Nebenläufigkeit

Läufe werden pro Sitzungsschlüssel (Sitzungs-Lane) und optional über eine globale Lane serialisiert, wodurch Werkzeug-/Sitzungs-Wettlaufsituationen verhindert werden. Nachrichtenkanäle wählen einen Warteschlangenmodus (steer/followup/collect/interrupt), der dieses Lane-System speist; siehe [Befehlswarteschlange](/de/concepts/queue).

Transkriptschreibvorgänge werden zusätzlich durch eine Sitzungsschreibsperre für die Sitzungsdatei geschützt. Die Sperre ist prozessbezogen und dateibasiert, sodass sie Schreibende erfasst, die die prozessinterne Warteschlange umgehen oder aus einem anderen Prozess stammen. Schreibende warten standardmäßig bis zu 60 Sekunden (Umgebungsüberschreibung `OPENCLAW_SESSION_WRITE_LOCK_ACQUIRE_TIMEOUT_MS`), bevor die Sitzung als beschäftigt gemeldet wird.

Sitzungsschreibsperren sind standardmäßig nicht wiedereintrittsfähig. Eine Hilfsfunktion, die absichtlich den Erwerb derselben Sperre verschachtelt und dabei einen einzigen logischen Schreibenden beibehält, muss dies mit `allowReentrant: true` ausdrücklich aktivieren.

## Vorbereitung von Sitzung und Arbeitsbereich

- Der Arbeitsbereich wird aufgelöst und erstellt; sandboxgeschützte Läufe können auf ein Sandbox-Arbeitsbereichsstammverzeichnis umgeleitet werden.
- Skills werden geladen (oder aus einem Snapshot wiederverwendet) und in die Umgebung und den Prompt eingefügt.
- Bootstrap-/Kontextdateien werden aufgelöst und in den System-Prompt eingefügt.
- Eine Sitzungsschreibsperre wird erworben und das Ziel für das Sitzungstranskript vorbereitet, bevor das Streaming beginnt. Jeder spätere Pfad zum Umschreiben, zur Compaction oder zur Kürzung des Transkripts muss dieselbe Sperre erwerben, bevor die SQLite-Transkriptzeilen geändert werden.

## Prompt-Zusammenstellung

Der System-Prompt wird aus dem Basis-Prompt von OpenClaw, dem Skills-Prompt, dem Bootstrap-Kontext und laufbezogenen Überschreibungen erstellt. Modellspezifische Limits und für die Compaction reservierte Tokens werden durchgesetzt. Unter [System-Prompt](/de/concepts/system-prompt) finden Sie, was das Modell sieht.

## Hooks

OpenClaw verfügt über zwei Hook-Systeme:

- **Interne Hooks** (Gateway-Hooks): ereignisgesteuerte Skripte für Befehle und Lebenszyklusereignisse.
- **Plugin-Hooks**: Erweiterungspunkte innerhalb des Agenten-/Werkzeuglebenszyklus und der Gateway-Pipeline.

### Interne Hooks (Gateway-Hooks)

- **`agent:bootstrap`**: Wird beim Erstellen der Bootstrap-Dateien ausgeführt, bevor der System-Prompt finalisiert wird. Verwenden Sie ihn, um Bootstrap-Kontextdateien hinzuzufügen oder zu entfernen.
- **Befehls-Hooks**: `/new`, `/reset`, `/stop` und andere Befehlsereignisse (siehe die Hooks-Dokumentation).

Einrichtung und Beispiele finden Sie unter [Hooks](/de/automation/hooks).

### Plugin-Hooks

Diese werden innerhalb des Agenten-Loops oder der Gateway-Pipeline ausgeführt:

| Hook                                                    | Ausführung                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `before_model_resolve`                                  | Vor der Sitzung (ohne `messages`), um Provider/Modell vor der Auflösung deterministisch zu überschreiben.                                                                                                                                                                                                |
| `before_prompt_build`                                   | Nach dem Laden der Sitzung (mit `messages`), um `prependContext`, `systemPrompt`, `prependSystemContext` oder `appendSystemContext` vor der Übermittlung einzufügen. Verwenden Sie `prependContext` für dynamischen Text pro Turn und die Systemkontextfelder für stabile Anweisungen, die in den System-Prompt gehören. |
| `before_agent_reply`                                    | Nach Inline-Aktionen und vor dem LLM-Aufruf. Ermöglicht einem Plugin, den Turn zu übernehmen und eine synthetische Antwort zurückzugeben oder ihn vollständig stummzuschalten.                                                                                                                                                                |
| `agent_end`                                             | Nach Abschluss, mit der endgültigen Nachrichtenliste und den Laufmetadaten.                                                                                                                                                                                                                             |
| `before_compaction` / `after_compaction`                | Beobachtet oder annotiert Compaction-Zyklen.                                                                                                                                                                                                                                                      |
| `before_tool_call` / `after_tool_call`                  | Fängt Werkzeugparameter/-ergebnisse ab.                                                                                                                                                                                                                                                              |
| `before_install`                                        | Nachdem die Installationsrichtlinie des Betreibers ausgeführt wurde, für bereitgestelltes Installationsmaterial von Skills/Plugins, wenn Plugin-Hooks im aktuellen Prozess geladen sind.                                                                                                                                                           |
| `tool_result_persist`                                   | Transformiert Werkzeugergebnisse synchron, bevor sie in ein OpenClaw-eigenes Sitzungstranskript geschrieben werden.                                                                                                                                                                                      |
| `message_received` / `message_sending` / `message_sent` | Hooks für eingehende und ausgehende Nachrichten.                                                                                                                                                                                                                                                         |
| `session_start` / `session_end`                         | Grenzen des Sitzungslebenszyklus.                                                                                                                                                                                                                                                               |
| `gateway_start` / `gateway_stop`                        | Gateway-Lebenszyklusereignisse.                                                                                                                                                                                                                                                                   |

Hook-Entscheidungsregeln für ausgehende/Werkzeug-Guards:

- `before_tool_call`: `{ block: true }` ist terminal und stoppt Handler mit niedrigerer Priorität. `{ block: false }` ist eine wirkungslose Operation und hebt eine vorherige Blockierung nicht auf.
- `before_install`: dieselbe Terminal-/Keine-Wirkung-Semantik wie oben. Verwenden Sie `security.installPolicy` und nicht `before_install` für betreiberseitige Entscheidungen zum Zulassen/Blockieren von Installationen, die CLI-Installations- und Aktualisierungspfade abdecken müssen.
- `message_sending`: `{ cancel: true }` ist terminal und stoppt Handler mit niedrigerer Priorität. `{ cancel: false }` ist eine wirkungslose Operation und hebt einen vorherigen Abbruch nicht auf.

Die Hook-API und Einzelheiten zur Registrierung finden Sie unter [Plugin-Hooks](/de/plugins/hooks).

Testumgebungen können diese Hooks anpassen. Die Codex-App-Server-Testumgebung behält OpenClaw-Plugin-Hooks als Kompatibilitätsvertrag für dokumentierte gespiegelte Oberflächen bei; native Codex-Hooks sind ein separater, systemnäherer Codex-Mechanismus.

## Streaming

- Assistenten-Deltas werden von der Agentenlaufzeit als `assistant`-Ereignisse gestreamt.
- Block-Streaming kann Teilantworten bei `text_end` oder `message_end` ausgeben.
- Das Streaming von Schlussfolgerungen kann als separater Stream oder in Blockantworten erfolgen.
- Informationen zu Aufteilung und Blockantwortverhalten finden Sie unter [Streaming](/de/concepts/streaming).

## Werkzeugausführung

- Ereignisse für Werkzeugstart/-aktualisierung/-ende werden im `tool`-Stream ausgegeben.
- Werkzeugergebnisse werden vor der Protokollierung/Ausgabe hinsichtlich Größe und Bildnutzdaten bereinigt.
- Sendungen durch Nachrichtenwerkzeuge werden verfolgt, um doppelte Assistentenbestätigungen zu unterdrücken.

## Antwortgestaltung

Endgültige Nutzdaten werden aus Assistententext (zuzüglich optionaler Schlussfolgerungen), Inline-Werkzeugzusammenfassungen (wenn ausführlich und zulässig) und Assistentenfehlertext bei Modellfehlern zusammengestellt.

- Das exakte Stille-Token `NO_REPLY` wird aus ausgehenden Nutzdaten herausgefiltert.
- Duplikate von Nachrichtenwerkzeugen werden aus der endgültigen Nutzdatenliste entfernt.
- Wenn keine darstellbaren Nutzdaten verbleiben und bei einem Werkzeug ein Fehler aufgetreten ist, wird ersatzweise eine Werkzeugfehlerantwort ausgegeben, sofern nicht bereits ein Nachrichtenwerkzeug eine für Benutzende sichtbare Antwort gesendet hat.

## Compaction und Wiederholungsversuche

Die automatische Compaction gibt `compaction`-Stream-Ereignisse aus und kann einen Wiederholungsversuch auslösen. Beim Wiederholungsversuch werden speicherinterne Puffer und Werkzeugzusammenfassungen zurückgesetzt, um doppelte Ausgaben zu vermeiden. Siehe [Compaction](/de/concepts/compaction).

## Ereignis-Streams

- `lifecycle`: Wird von `subscribeEmbeddedAgentSession` ausgegeben (und ersatzweise von `agentCommand`).
- `assistant`: Gestreamte Deltas aus der Agentenlaufzeit.
- `tool`: Gestreamte Werkzeugereignisse aus der Agentenlaufzeit.

Das Gateway projiziert Lebenszyklusereignisse sowie Start-/Terminalereignisse von Werkzeugen in das begrenzte,
rein metadatenbasierte [Audit-Ledger](/de/cli/audit). Diese Projektion zeichnet Herkunft und
Ergebniscodes auf, ohne Prompts, Nachrichten, Werkzeugargumente, Werkzeugergebnisse
oder Rohfehler aus dem Transkript-/Laufzeitpfad zu kopieren.

## Verarbeitung von Chatkanälen

Assistenten-Deltas werden in Chat-`delta`-Nachrichten gepuffert. Bei **Lebenszyklusende/-fehler** wird ein Chat-`final` ausgegeben.

## Zeitlimits

| Zeitüberschreitung                               | Standardwert                            | Hinweise                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------------------------------------------------ | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent.wait`                                     | 30s                                    | Nur Warten; der Parameter `timeoutMs` überschreibt diesen Wert. Stoppt den zugrunde liegenden Lauf nicht.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Agent-Laufzeit (`agents.defaults.timeoutSeconds`) | 172800s (48h)                          | Wird durch den Abbruch-Timer von `runEmbeddedAgent` erzwungen. Legen Sie `0` für ein unbegrenztes Laufbudget fest; Watchdogs für die Aktivität des Modellstreams gelten weiterhin.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Watchdog des CLI-Backends bei ausbleibender Ausgabe | wird für jeden neuen/fortgesetzten CLI-Lauf berechnet | Von der Agent-Laufzeit getrennt und im Besitz des registrierten Backend-Plugins. Eine CLI-interne Hintergrundaufgabe nutzt denselben übergeordneten Unterprozess und läuft nicht über eine allgemeine Agent-Zeitüberschreitung hinaus weiter.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Isolierter Agent-Durchlauf von Cron               | im Besitz von Cron                     | Der Scheduler startet bei Ausführungsbeginn einen eigenen Timer, bricht den Lauf zum konfigurierten Stichtag ab und führt anschließend eine begrenzte Bereinigung durch, bevor er die Zeitüberschreitung erfasst, damit eine veraltete untergeordnete Sitzung die Lane nicht blockiert halten kann.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Modell-Leerlaufzeitüberschreitung                 | Cloud 120s; selbst gehostet 300s       | OpenClaw bricht eine Modellanfrage ab, wenn vor Ablauf des Leerlauffensters keine Antwort-Chunks eintreffen. `models.providers.<id>.timeoutSeconds` verlängert diesen Leerlauf-Watchdog für langsame lokale/selbst gehostete Provider, bleibt jedoch durch jede niedrigere endliche Zeitüberschreitung von `agents.defaults.timeoutSeconds` oder eine laufspezifische Zeitüberschreitung begrenzt, da diese den gesamten Agent-Lauf steuern. Bei unbegrenzten Laufbudgets bleibt der Leerlauf-Watchdog der Provider-Klasse weiterhin aktiv. Durch Cron ausgelöste Cloud-Modellläufe ohne explizite Modell-/Agent-Zeitüberschreitung verwenden denselben Standardwert; bei einer expliziten Cron-Laufzeitüberschreitung werden Stillstände des Cloud-Modellstreams auf 60s begrenzt, damit konfigurierte Modell-Fallbacks noch vor dem äußeren Cron-Stichtag ausgeführt werden können. Durch Cron ausgelöste Läufe auf tatsächlich lokalen Endpunkten (Loopback/private baseUrl) behalten die lokale Leerlauf-Deaktivierung bei; selbst gehostete Provider mit Netzwerk-baseUrls erhalten den impliziten Watchdog von 300s. Bei einer expliziten Cron-Laufzeitüberschreitung werden lokale/selbst gehostete Stillstände auf diese Zeitüberschreitung begrenzt. Legen Sie `models.providers.<id>.timeoutSeconds` für langsame lokale Provider fest. |
| Zeitüberschreitung für HTTP-Anfragen des Providers | `models.providers.<id>.timeoutSeconds` | Deckt Verbindungsaufbau, Header, Body, SDK-Anfragezeitüberschreitung, Abbruchbehandlung von guarded-fetch und den Leerlauf-Watchdog des Modellstreams für diesen Provider ab. Verwenden Sie diese Einstellung für langsame lokale/selbst gehostete Provider (beispielsweise Ollama), bevor Sie die Zeitüberschreitung der gesamten Agent-Laufzeit erhöhen; halten Sie die Agent-/Laufzeitüberschreitung mindestens ebenso hoch, wenn die Modellanfrage länger ausgeführt werden muss.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |

### Diagnose festgefahrener Sitzungen

Bei aktivierter Diagnose klassifiziert ein integrierter Schwellenwert von zwei Minuten lang laufende `processing`-Sitzungen, bei denen keine Antwort sowie kein Werkzeug-, Status-, Block- oder ACP-Fortschritt beobachtet wurde:

- Aktive eingebettete Läufe, Modellaufrufe und Werkzeugaufrufe werden als `session.long_running` gemeldet. Zugeordnete stille Modellaufrufe bleiben bis zum Abbruchschwellenwert `session.long_running`, damit langsame oder nicht streamende Provider nicht zu früh als festgefahren gekennzeichnet werden.
- Aktive Arbeit ohne kürzlichen Fortschritt wird als `session.stalled` gemeldet. Zugeordnete Modellaufrufe wechseln beim oder nach dem Abbruchschwellenwert zu `session.stalled`; veraltete Modell-/Werkzeugaktivität ohne Eigentümer wird nicht als lang laufend verborgen.
- `session.stuck` ist für wiederherstellbare veraltete Sitzungsbuchführung reserviert, einschließlich inaktiver Sitzungen in der Warteschlange mit veralteter Modell-/Werkzeugaktivität ohne Eigentümer.

Der Abbruchschwellenwert beträgt mindestens 5 Minuten und das Dreifache des Warnschwellenwerts. Die Bereinigung veralteter Sitzungsbuchführung gibt die betroffene Sitzungs-Lane unmittelbar frei, nachdem die Wiederherstellungsprüfungen bestanden wurden; festgefahrene eingebettete Läufe werden erst nach dem Abbruchschwellenwert abgebrochen und geleert, sodass Arbeit in der Warteschlange fortgesetzt wird, ohne lediglich langsame Läufe abzubrechen. Die Wiederherstellung gibt strukturierte Ergebnisse für Anforderung und Abschluss aus; der Diagnosestatus wird nur dann als inaktiv markiert, wenn dieselbe Verarbeitungsgeneration noch aktuell ist, und wiederholte `session.stuck`-Diagnosen verwenden ein zunehmendes Warteintervall, solange die Sitzung unverändert bleibt.

## Mögliche Gründe für eine vorzeitige Beendigung

- Agent-Zeitüberschreitung (Abbruch)
- AbortSignal (Abbrechen)
- Gateway-Verbindungsabbruch oder RPC-Zeitüberschreitung
- Zeitüberschreitung von `agent.wait` (nur Warten, stoppt den Agent nicht)

## Verwandte Themen

- [Werkzeuge](/de/tools) – verfügbare Agent-Werkzeuge
- [Hooks](/de/automation/hooks) – ereignisgesteuerte Skripte, die durch Ereignisse im Agent-Lebenszyklus ausgelöst werden
- [Compaction](/de/concepts/compaction) – wie lange Unterhaltungen zusammengefasst werden
- [Ausführungsgenehmigungen](/de/tools/exec-approvals) – Genehmigungsschranken für Shell-Befehle
- [Denken](/de/tools/thinking) – Konfiguration der Denk-/Schlussfolgerungsebene
