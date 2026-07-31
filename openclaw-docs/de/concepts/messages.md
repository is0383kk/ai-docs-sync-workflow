---
read_when:
    - Erklärung, wie eingehende Nachrichten zu Antworten werden
    - Erläuterung von Sitzungen, Warteschlangenmodi oder Streaming-Verhalten
    - Dokumentation der Sichtbarkeit von Schlussfolgerungen und der Auswirkungen auf die Nutzung
summary: Nachrichtenfluss, Sitzungen, Warteschlangen und Sichtbarkeit von Schlussfolgerungen
title: Nachrichten
x-i18n:
    generated_at: "2026-07-26T17:48:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e42bed834e9a57fb8a248c8654b75ea9977928582f68a83859cf6c16ed0b6bf5
    source_path: concepts/messages.md
    workflow: 16
---

Eingehende Nachrichten durchlaufen Routing, Deduplizierung/Entprellung, einen Agentenlauf und die ausgehende Zustellung:

```text
Eingehende Nachricht
  -> Routing/Bindungen -> Sitzungsschlüssel
  -> Deduplizierung + Entprellung
  -> Warteschlange (wenn bereits ein Lauf aktiv ist)
  -> Agentenlauf (Streaming + Tools)
  -> ausgehende Antworten (Kanallimits + Aufteilung)
```

Wichtige Konfigurationsbereiche:

- `messages.*` für Präfixe, Warteschlangen, die Entprellung eingehender Nachrichten und das Gruppenverhalten.
- `agents.defaults.*` für Block-Streaming, Aufteilung und Standardwerte für stille Antworten.
- Kanalspezifische Überschreibungen (`channels.telegram.*`, `channels.whatsapp.*` usw.) für kanalspezifische Limits und Streaming-Schalter.

Das vollständige Schema finden Sie unter [Konfiguration](/de/gateway/configuration).

## Deduplizierung eingehender Nachrichten

Kanäle können dieselbe Nachricht nach einer erneuten Verbindung erneut zustellen. OpenClaw führt einen In-Memory-Cache, dessen Schlüssel aus dem Agentenbereich, der Kanalroute (Kanal + Kommunikationspartner + Konto + Thread) und der Nachrichten-ID besteht, sodass eine erneut zugestellte Nachricht keinen zweiten Agentenlauf auslöst. Der Cache-Eintrag läuft nach 20 Minuten oder beim Erreichen von 5000 erfassten Einträgen ab, je nachdem, was zuerst eintritt.

## Entprellung eingehender Nachrichten

Schnell aufeinanderfolgende Textnachrichten desselben Absenders können über `messages.inbound` zu einem Agentendurchlauf zusammengefasst werden. Die Entprellung gilt jeweils pro Kanal + Unterhaltung und verwendet die neueste Nachricht für Antwort-Threading/IDs.

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,
      byChannel: {
        discord: 1500,
        slack: 1500,
        whatsapp: 5000,
      },
    },
  },
}
```

- Die Entprellung gilt nur für reine Textnachrichten; Medien/Anhänge werden sofort weitergeleitet.
- Steuerbefehle (Stopp/Abbruch/Status usw.) umgehen die Entprellung, damit sie sofort weitergeleitet werden.
- Standardmäßig deaktiviert: `messages.inbound.debounceMs` hat keinen integrierten Standardwert, sodass die Entprellung erst aktiviert wird, wenn Sie sie festlegen (global oder pro Kanal).
- iMessage folgt derselben allgemeinen Entprellungsrichtlinie. `imsg` 0.13.1 und neuer fasst durch Apple-URL-Vorschauen aufgeteilte Sendungen zusammen, bevor OpenClaw sie empfängt, sodass keine iMessage-spezifische Entprellungseinstellung erforderlich ist.

## Sitzungen und Geräte

Sitzungen werden vom Gateway verwaltet, nicht von Clients.

- Direktchats werden im Hauptsitzungsschlüssel des Agenten zusammengeführt.
- Gruppen/Kanäle erhalten eigene Sitzungsschlüssel.
- Der Sitzungsspeicher und die Transkripte befinden sich auf dem Gateway-Host.

Mehrere Geräte/Kanäle können derselben Sitzung zugeordnet sein, der Verlauf wird jedoch nicht vollständig mit jedem Client zurücksynchronisiert. Verwenden Sie für lange Unterhaltungen ein primäres Gerät, um auseinanderlaufenden Kontext zu vermeiden. Die Control UI und die TUI zeigen stets das vom Gateway bereitgestellte Sitzungstranskript und sind daher die maßgebliche Quelle.

Details: [Sitzungsverwaltung](/de/concepts/session).

## Prompt-Inhalte und Verlaufskontext

Kanal-Plugins füllen im eingehenden Kontext mehrere Textfelder aus, geordnet von der höchsten zur niedrigsten Priorität:

| Feld              | Zweck                                                                                                                |
| ----------------- | -------------------------------------------------------------------------------------------------------------------- |
| `BodyForAgent`    | An das Modell gerichteter Text für den aktuellen Durchlauf. Greift auf `CommandBody` / `RawBody` / `Body` zurück, wenn nicht festgelegt.        |
| `BodyForCommands` | Bereinigter Text für die Analyse von Direktiven/Befehlen. Greift auf `CommandBody` / `RawBody` / `Body` zurück, wenn nicht festgelegt. |
| `CommandBody`     | Veralteter Zwischeninhalt; verwenden Sie vorzugsweise `BodyForCommands`.                                                         |
| `RawBody`         | Veralteter Alias für `CommandBody`.                                                                         |
| `Body`            | Veralteter Prompt-Inhalt; kann Kanalumschläge und Verlaufshüllen enthalten.                                     |

Wenn ein Kanal Verlauf bereitstellt, umschließt er ihn mit:

- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

Bei Nicht-Direktchats (Gruppen/Kanälen/Räumen) wird dem aktuellen Nachrichteninhalt die Absenderbezeichnung vorangestellt, entsprechend dem für Verlaufseinträge verwendeten Stil. Das Entfernen von Direktiven gilt nur für den Abschnitt der aktuellen Nachricht, sodass der Verlauf unverändert bleibt. Kanäle, die den Verlauf umschließen, sollten `BodyForCommands` (oder die veralteten `CommandBody` / `RawBody`) auf den ursprünglichen Nachrichtentext setzen und `Body` als kombinierten Prompt beibehalten.

Verlaufspuffer enthalten nur ausstehende Nachrichten: Sie umfassen Gruppennachrichten, die keinen Lauf ausgelöst haben (beispielsweise Nachrichten, die aufgrund einer Erwähnungssperre nicht verarbeitet wurden), und schließen Nachrichten aus, die bereits im Sitzungstranskript enthalten sind. Strukturierter Verlauf sowie Antwort-, Weiterleitungs- und Kanalmetadaten werden bei der Prompt-Zusammenstellung als nicht vertrauenswürdige Kontextblöcke der Benutzerrolle dargestellt.

Konfigurieren Sie die Verlaufsgröße mit `messages.groupChat.historyLimit` (globaler Standardwert) oder kanalspezifischen Überschreibungen wie `channels.slack.historyLimit` und `channels.telegram.accounts.<id>.historyLimit` (setzen Sie `0`, um sie zu deaktivieren).

## Metadaten von Tool-Ergebnissen

`content` eines Tool-Ergebnisses ist das für das Modell sichtbare Ergebnis; `details` sind Laufzeitmetadaten für die UI-Darstellung, Diagnose, Medienzustellung und Plugins.

- `toolResult.details` wird vor der erneuten Wiedergabe durch den Provider und vor der Eingabe für die Compaction entfernt.
- Persistierte Sitzungstranskripte behalten nur begrenzte `details`; übergroße Metadaten werden durch eine kompakte, mit `persistedDetailsTruncated: true` gekennzeichnete Zusammenfassung ersetzt.
- Plugins und Tools sollten Text, den das Modell lesen muss, in `content` ablegen, nicht ausschließlich in `details`.

## Warteschlangen und Folgenachrichten

Wenn bereits ein Lauf aktiv ist, werden eingehende Nachrichten standardmäßig in diesen eingespeist. `messages.queue` steuert den Modus:

| Modus             | Verhalten                                                |
| ----------------- | -------------------------------------------------------- |
| `steer` (Standard) | Den neuen Prompt in den aktiven Lauf einspeisen.         |
| `followup`        | Die Nachricht nach Abschluss des aktiven Laufs ausführen. |
| `collect`         | Kompatible Nachrichten zu einem späteren Durchlauf bündeln. |
| `interrupt`       | Den aktiven Lauf abbrechen und dann den neuesten Prompt starten. |

Die Warteschlange verwendet eine integrierte Entprellung von 500ms für die Bündelung von Steuerungs-, Folge- und Sammelnachrichten. `messages.queue.cap` ist standardmäßig auf 20 Nachrichten in der Warteschlange begrenzt und `messages.queue.drop` verwendet standardmäßig `summarize` (`old` und `new` sind ebenfalls verfügbar). Konfigurieren Sie kanalspezifische Überschreibungen über `messages.queue.byChannel` und `messages.queue.debounceMsByChannel`.

Details: [Befehlswarteschlange](/de/concepts/queue) und [Steuerungswarteschlange](/de/concepts/queue-steering).

## Eigentümerschaft von Kanalläufen

Kanal-Plugins können die Reihenfolge beibehalten, Eingaben entprellen und Transport-Gegendruck anwenden, bevor eine Nachricht in die Sitzungswarteschlange gelangt. Sie sollten den Agentendurchlauf selbst nicht mit einem separaten Timeout versehen. Sobald eine Nachricht an eine Sitzung weitergeleitet wurde, steuert der Lebenszyklus von Sitzung, Tool und Laufzeit lang andauernde Vorgänge, sodass alle Kanäle langsame Durchläufe einheitlich melden und sich davon erholen.

## Streaming, Aufteilung und Bündelung

Block-Streaming sendet Teilantworten, während das Modell Textblöcke erzeugt; die Aufteilung berücksichtigt die Textlimits des Kanals und vermeidet das Trennen eingezäunter Codeblöcke.

- `agents.defaults.blockStreamingDefault` (`on|off`, Standard `off`)
- `agents.defaults.blockStreamingBreak` (`text_end|message_end`)
- `agents.defaults.blockStreamingChunk` (`minChars|maxChars|breakPreference`)
- `agents.defaults.blockStreamingCoalesce` (leerlaufbasierte Bündelung)
- `agents.defaults.humanDelay` (menschenähnliche Pause zwischen Blockantworten)
- Kanalspezifische Überschreibungen: `*.streaming.block.enabled` und `*.streaming.block.coalesce` bei gebündelten Kanälen; veraltete flache Schlüssel werden von `openclaw doctor --fix` migriert. Block-Streaming ist auf jedem Kanal einschließlich Telegram deaktiviert, sofern es nicht ausdrücklich aktiviert wird. QQ Bot ist die Ausnahme: Er verfügt über keine `streaming.block`-Schlüssel und streamt Blockantworten, sofern `channels.qqbot.streaming.mode` nicht `"off"` ist.

Details: [Streaming + Aufteilung](/de/concepts/streaming).

## Sichtbarkeit von Schlussfolgerungen und Tokens

- `/reasoning on|off|stream` steuert die Sichtbarkeit.
- Inhalte von Schlussfolgerungen zählen weiterhin zur Token-Nutzung, wenn das Modell sie erzeugt.
- Telegram unterstützt das Streamen von Schlussfolgerungen in eine vorübergehende Entwurfsblase, die nach der endgültigen Zustellung gelöscht wird; verwenden Sie `/reasoning on` für eine persistente Ausgabe der Schlussfolgerungen.

Details: [Denk- und Schlussfolgerungsdirektiven](/de/tools/thinking) und [Token-Nutzung](/de/reference/token-use).

## Präfixe, Threading und Antworten

- Präfixe ausgehender Nachrichten befinden sich unter `channels.<channel>.responsePrefix` und `channels.<channel>.accounts.<id>.responsePrefix`. Kontowerte haben Vorrang. Doctor kopiert den globalen Rückfallwert in konfigurierte Kanalblöcke, wenn diese kanonischen Felder nicht festgelegt sind; `messages.responsePrefix` bleibt als Rückfallwert für implizite und benutzerdefinierte Kanäle erhalten.
- Antwort-Threading über `replyToMode` und kanalspezifische Standardwerte.

Details: [Konfiguration](/de/gateway/config-agents#messages) und Kanaldokumentation.

## Stille Antworten

Das stille Token `NO_REPLY` (Groß-/Kleinschreibung wird nicht berücksichtigt, sodass auch `no_reply` übereinstimmt) bedeutet „keine für den Benutzer sichtbare Antwort zustellen“. Wenn für einen Durchlauf zugleich Tool-Medien ausstehen, etwa erzeugtes TTS-Audio, entfernt OpenClaw den stillen Text, stellt den Medienanhang jedoch weiterhin zu.

Die Stillerichtlinie wird nach Unterhaltungstyp aufgelöst:

- Direkte Unterhaltungen erhalten niemals `NO_REPLY`-Prompt-Anweisungen. Wenn ein direkter Lauf versehentlich nur ein stilles Token zurückgibt, unterdrückt OpenClaw es, anstatt es umzuschreiben oder zuzustellen.
- Gruppen/Kanäle erlauben standardmäßig Stille. Im Modus `message_tool` für sichtbare Antworten bedeutet Stille, dass das Modell `message(action=send)` nicht aufruft.
- Interne Orchestrierung erlaubt standardmäßig Stille.

Die Standardwerte befinden sich unter `agents.defaults.silentReply`; `surfaces.<id>.silentReply` kann die Gruppen-/internen Richtlinien pro Oberfläche überschreiben.

OpenClaw verwendet stille Antworten außerdem bei allgemeinen internen Runner-Fehlern in Nicht-Direktchats, damit Gruppen/Kanäle keine standardisierten Gateway-Fehlermeldungen sehen. Klassifizierte Fehler mit benutzerorientierten Wiederherstellungshinweisen, etwa Hinweise auf fehlende Authentifizierung, Ratenbegrenzung oder Überlastung, können weiterhin zugestellt werden. Direktchats zeigen standardmäßig kompakte Fehlermeldungen; unverarbeitete Runner-Details werden nur angezeigt, wenn `/verbose full` aktiviert ist.

Reine stille Antworten werden auf allen Oberflächen verworfen, sodass übergeordnete Sitzungen ruhig bleiben, anstatt Sentinel-Text in ersatzweise Konversation umzuschreiben.

## Verwandte Themen

- [Refaktorierung des Nachrichtenlebenszyklus](/de/concepts/message-lifecycle-refactor) - angestrebtes robustes Sende- und Empfangsdesign
- [Streaming](/de/concepts/streaming) - Nachrichtenzustellung in Echtzeit
- [Wiederholungsversuche](/de/concepts/retry) - Verhalten bei Wiederholungsversuchen der Nachrichtenzustellung
- [Warteschlange](/de/concepts/queue) - Warteschlange für die Nachrichtenverarbeitung
- [Kanäle](/de/channels) - Integrationen für Messaging-Plattformen
