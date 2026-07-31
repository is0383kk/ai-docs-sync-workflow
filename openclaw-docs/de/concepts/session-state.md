---
read_when:
    - Sie möchten, dass Agenten bemerken, wenn Menschen oder andere Agenten eine Sitzung ohne ihr Wissen ändern
    - Sie debuggen Statusänderungsbenachrichtigungen, Watch-Cursor oder session_status changesSince
    - Sie möchten verstehen, wie übergeordnete Agenten mit untergeordneten Sitzungen synchronisiert bleiben
sidebarTitle: Session state awareness
summary: 'Protokoll dauerhafter Sitzungsstatussignale: Statusversionen, Überwachungsprozesse, Hinweise auf veraltete Statusdaten und Abgleich'
title: Sitzungsstatusbewusstsein
x-i18n:
    generated_at: "2026-07-26T18:21:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb4126a0802e1ca4418f225c792490493a78886089b81c3b4567f72090ce34f4
    source_path: concepts/session-state.md
    workflow: 16
---

Wenn mehrere Sitzungen am selben Problem arbeiten — ein Manager, der Aufgaben an untergeordnete Sitzungen delegiert, ein Mensch, der direkt in eine Worker-Sitzung wechselt, oder zwei Agenten, die sich über [`sessions_send`](/de/concepts/session-tool) koordinieren —, baut jede Sitzung Annahmen über die anderen auf. Diese Annahmen sind überholt, sobald ein anderer Akteur eingreift. Die Sitzungsstatuswahrnehmung ist der Mechanismus, der den Eingriff erkennt, die betroffene Sitzung einmalig benachrichtigt und ihr eine kostengünstige Möglichkeit bietet, sich vor dem Handeln auf den aktuellen Stand zu bringen.

Drei Komponenten arbeiten zusammen:

1. Ein **dauerhaftes Signalprotokoll** zeichnet ausgewählte Statusänderungen pro Sitzung auf.
2. **Watcher** verwalten Cursor pro Ziel und erhalten eine einzelne zusammengefasste Benachrichtigung über einen veralteten Status.
3. Die **Reconciliation** ruft über `session_status` mit `changesSince` das exakte Delta ab.

## Das Signalprotokoll

OpenClaw fügt der gemeinsam genutzten Statusdatenbank (`session_state_events`) ein typisiertes Ereignis hinzu, wenn sich eine überwachte Sitzung wesentlich ändert. Ereignisse enthalten Metadaten und eine einzeilige Zusammenfassung — niemals Nachrichteninhalte.

| Art                    | Aufgezeichnet, wenn                                      | Benachrichtigt Watcher |
| ---------------------- | -------------------------------------------------------- | ---------------------- |
| `human_direct_message` | Ein Mensch sendet direkt einen Turn an eine überwachte Sitzung | Ja                 |
| `upstream_missing`     | Die Upstream-Quelle einer übernommenen Sitzung verschwindet | Ja                 |
| `goal_changed`         | Der Zielstatus der Sitzung erstellt, aktualisiert oder gelöscht wird | Ja         |
| `child_spawned`        | Eine untergeordnete Agenten- oder ACP-Sitzung erstellt wird | Nein (initialisiert Cursor) |
| `run_completed`        | Ein untergeordneter Lauf erfolgreich endet              | Nein (nur Protokoll)   |
| `run_failed`           | Ein untergeordneter Lauf fehlschlägt, das Zeitlimit überschreitet oder abgebrochen wird | Nein (nur Protokoll) |
| `compacted`            | Der Verlauf der Sitzung komprimiert wird                 | Nein (nur Protokoll)   |
| `adopted`              | Eine Katalogsitzung in OpenClaw übernommen wird          | Nein (nur Protokoll)   |

Jedes Ereignis nennt seinen Akteur (`human`, `agent` oder `system`). Abgebrochene untergeordnete Läufe und solche mit Zeitüberschreitung werden als Fehlschläge aufgezeichnet, wobei das genaue Ergebnis (`cancelled`, `timeout` oder `error`) in den Ereignisdaten erhalten bleibt.

Die **Statusversion** einer Sitzung ist einfach die höchste Sequenznummer in ihrem Protokoll. Sie wird in einem dauerhaften sitzungsspezifischen Kopfdatensatz verfolgt, der die Bereinigung überdauert. `sessions_list`-Zeilen enthalten `stateVersion`, wenn für eine Sitzung Änderungen protokolliert wurden; `session_status` gibt sie immer zurück.

Nur protokollierte Arten dienen dem Reconciliation-Verlauf, nicht der Benachrichtigung: Die reguläre Übermittlung abgeschlossener untergeordneter Läufe bleibt Aufgabe der [Ankündigungen untergeordneter Agenten](/de/tools/subagents), und das Signalprotokoll dupliziert sie niemals.

## Watcher

Ein Watcher ist eine Sitzung, die einen Cursor (`session_watch_cursors`) auf einem Ziel verwaltet. Cursor stammen aus zwei Quellen:

- **Implizit (Spawn-Kanten).** Wenn eine Sitzung einen untergeordneten Agenten oder eine untergeordnete ACP-Sitzung erzeugt, wird der Cursor der übergeordneten Sitzung automatisch mit der Spawn-Version der untergeordneten Sitzung initialisiert. Übergeordnete Sitzungen abonnieren niemals manuell.
- **Explizit (`sessions_send watch: true`).** Jeder Koordinator kann ein nicht erzeugtes Ziel überwachen: Übergeben Sie `watch: true` an `sessions_send`. Nachdem der Versand erfolgreich erfolgt ist, wird der Absender als Watcher der Sitzung registriert, die die Nachricht tatsächlich empfangen hat. Die Registrierung beginnt bei der aktuellen Statusversion des Ziels — der vorherige Verlauf erzeugt niemals Benachrichtigungen. Das Werkzeugergebnis meldet `watched: true|false`, wenn der Parameter gesetzt wurde.

Die Identität eines Watchers muss ein agentenqualifizierter Sitzungsschlüssel sein. Unter `session.scope="global"` ist der gemeinsam genutzte Schlüssel `global` über mehrere Agenten hinweg mehrdeutig. Daher erhalten solche Sitzungen das dauerhafte Protokoll und `changesSince`, aber keine proaktiven Benachrichtigungen.

Überwachungen bereinigen sich selbst: Cursorzeilen laufen gemäß der Aufbewahrungsdauer des Signalprotokolls ab, werden beim Zurücksetzen der Watcher-Sitzung entfernt und zusammen mit einer der beiden Sitzungen gelöscht. In v1 gibt es keinen Befehl zum Beenden der Überwachung.

Überwachte Sitzungen, die aus einem Sitzungskatalog übernommen wurden, werden in einem festen Intervall auf direkte menschliche Aktivität in der Upstream-Quelle geprüft. Erkannte Aktivität durchläuft dasselbe Signalprotokoll und denselben Watcher-Ablauf wie andere direkte menschliche Turns.

Wenn die Upstream-Quelle einer übernommenen Sitzung extern gelöscht wird, erzeugen drei aufeinanderfolgende erfolglose Prüfungen (etwa drei Monitor-Takte) ein einzelnes `upstream_missing`-Signal für ihre Watcher und entfernen die Upstream-Verknüpfung. Wird die Katalogsitzung erneut fortgesetzt, entsteht eine neue Verknüpfung.

## Benachrichtigungen: eine statt vieler

Wenn ein benachrichtigungsfähiges Ereignis eintrifft und der Cursor eines Watchers zurückliegt, erhält der Watcher bei seinem nächsten Turn eine einzelne Systembenachrichtigung:

```
Sitzung "agent:main:subagent:child" wurde geändert (anderer Akteur). Führen Sie vor dem Handeln eine Reconciliation durch: session_status sessionKey "agent:main:subagent:child" changesSince 12.
```

Watcher der Hauptsitzung werden außerdem sofort durch einen Heartbeat-Wake geweckt; verschachtelte Watcher untergeordneter Agenten erhalten die Benachrichtigung bei ihrem nächsten Turn.

Das Protokoll ist bewusst gegen Benachrichtigungsfluten ausgelegt:

- **Eine ausstehende Benachrichtigung pro Watcher-Ziel-Paar.** Der Benachrichtigungstext bleibt im ausstehenden Zustand byteidentisch, und die Systemereigniswarteschlange dedupliziert anhand dieses Textes. Daher erzeugen selbst zwanzig schnelle Änderungen am selben Ziel nur eine einzige Zeile im Prompt des Watchers.
- **Eingefrorene Hochwassermarke.** Der Cursor friert seine benachrichtigte Position ein, wenn eine Benachrichtigung in die Warteschlange gestellt wird. Weitere wesentliche Ereignisse erhöhen nur die wesentliche Hochwassermarke; sie lösen keine erneute Benachrichtigung aus.
- **Bestätigung beim Entnehmen, erneutes Öffnen nur bei dazwischenliegender Arbeit.** Wenn der Turn des Watchers die Benachrichtigung verarbeitet, wird der Cursor weitergesetzt. Sind zwischen dem Einreihen und dem Entnehmen weitere wesentliche Ereignisse eingetroffen, wird für den Rest genau eine neue Benachrichtigung geöffnet.
- **Selbstunterdrückung.** Ein Watcher wird niemals über Ereignisse benachrichtigt, die er selbst verursacht hat.
- **Wiederherstellung nach einem Neustart.** Ausstehende Benachrichtigungen befinden sich in einer In-Memory-Warteschlange; nach einem Gateway-Neustart materialisiert eine Startprüfung sie anhand der dauerhaften Cursor erneut.

## Reconciliation durchführen

Die Benachrichtigung teilt dem Watcher genau mit, was zu tun ist. `session_status` mit `changesSince: <version>` gibt die typisierten Ereignisse nach dieser Version zurück (bis zu 200), ohne Cursor weiterzusetzen:

```json
{
  "stateVersion": 19,
  "stateChanges": {
    "events": [
      {
        "sequence": 14,
        "kind": "human_direct_message",
        "actorType": "human",
        "summary": "menschliche Nachricht über Telegram"
      },
      { "sequence": 19, "kind": "goal_changed", "actorType": "human", "summary": "Ziel aktualisiert" }
    ],
    "historyGap": false
  }
}
```

`historyGap: true` bedeutet, dass die angeforderte Version älter als der aufbewahrte Verlauf ist — aktualisieren Sie den gesamten Sitzungsstatus (`sessions_history`, `session_status`), statt die Antwort als exaktes Delta zu behandeln. Das Lückensignal ist exakt: Es stammt aus einer sitzungsspezifischen Hochwassermarke für Bereinigungen und wird nicht aus der Sequenzarithmetik abgeleitet.

## Speicherung und Grenzen

Der Verlauf befindet sich in der gemeinsam genutzten Statusdatenbank und ist auf 30 Tage und 50.000 Zeilen begrenzt; sitzungsspezifische Kopfdatensätze bleiben nach der Bereinigung monoton. Die Aufzeichnung erfolgt nach bestem Bemühen — ein fehlgeschlagenes Anhängen wird protokolliert und lässt den auslösenden Turn niemals fehlschlagen —, daher ist `stateVersion` ein Kopfdatensatz des Signalprotokolls und keine transaktionale Change-Data-Capture-Version.

Aktuelle Grenzen:

- Die Zustellung von Benachrichtigungen setzt voraus, dass ein Gateway-Prozess die gemeinsam genutzte Statusdatenbank verwaltet. Mehrere Gateways teilen sich das dauerhafte Protokoll und `changesSince`, aber v1 überträgt Benachrichtigungen nicht prozessübergreifend.
- Compaction-Ereignisse decken die Compaction-Verantwortlichen der eingebetteten Runtime ab; eine ausschließlich im nativen Harness ausgeführte Compaction wird nicht vollständig protokolliert.
- Detaillierte Nutzdaten zu Abbruchergebnissen werden derzeit von untergeordneten ACP-Läufen erzeugt; Abbrüche nativer untergeordneter Agenten werden als generische Fehlschläge dargestellt.
- Die Erkennung von Upstream-Selbstechos vergleicht normalisierten Benutzertext. Ein externer Prompt, der mit einer der 10 neuesten benutzerseitigen OpenClaw-Nachrichten der Sitzung übereinstimmt, wird als Selbstecho behandelt.
- Eine einzelne lokale Claude-JSONL-Zeile, die größer als die Scanbegrenzung von 1 MiB pro Intervall ist, blockiert in v1 den Cursor dieser Sitzung; nicht klassifizierte Bytes werden niemals übersprungen.
- Claude-Prüfungen auf gekoppelten Nodes klassifizieren pro Intervall die neuesten 50 Transkriptelemente. Größere Aktivitätsschübe können außerhalb des Scanfensters von v1 liegen.
- Claude-Verlaufsabfragen auf gekoppelten Nodes stellen kein eindeutiges Ergebnis für einen nicht gefundenen Thread bereit. Daher werden Remote-Löschungen in Claude in v1 nicht als `upstream_missing` klassifiziert.
- Katalogsitzungen, die nicht übernommen wurden, bleiben in v1 außerhalb der Wahrnehmungsschicht.
- Sitzungen, die vor Einführung dieser Funktion übernommen wurden, besitzen keine Upstream-Verknüpfung; setzen Sie sie einmal über den Katalog fort, um die Upstream-Überwachung zu starten.
- Upstream-Verknüpfungen setzen voraus, dass jeder Schlüssel einer übernommenen Sitzung genau einem zuständigen Agenten zugeordnet ist (bei der Übernahme wird der Standardagent des Speichers verwendet). Die Übernahme desselben externen Threads durch mehrere Agenten wird in v1 nicht überwacht.

## Verwandte Themen

- [Sitzungswerkzeuge](/de/concepts/session-tool) — `sessions_send`, `session_status`, `sessions_list`
- [Untergeordnete Agenten](/de/tools/subagents) — Spawn-Kanten und Abschlussankündigungen
- [Heartbeat](/de/gateway/heartbeat) — wie Benachrichtigungen in der Warteschlange Hauptsitzungen wecken
- [Sitzungsverwaltung](/de/concepts/session) — Sitzungsschlüssel, Geltungsbereiche und Lebenszyklus
