---
read_when:
    - Erläuterung des Verhaltens von Steer, während ein Agent Tools verwendet
    - Ändern des Verhaltens der Warteschlange für aktive Ausführungen oder der Integration der Laufzeitsteuerung
    - Vergleich von Steering mit den Warteschlangenmodi Followup, Collect und Interrupt
summary: Wie die Active-Run-Steuerung Nachrichten an Laufzeitgrenzen in die Warteschlange einreiht
title: Steuerungswarteschlange
x-i18n:
    generated_at: "2026-07-26T18:25:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 131f04f19934b9b1f6dd8ffb2cf2428950c319483abdc2ccdecec741809cda2a
    source_path: concepts/queue-steering.md
    workflow: 16
---

Wenn eine normale Eingabeaufforderung eintrifft, während ein Sitzungsdurchlauf bereits eine Ausgabe streamt und der Warteschlangenmodus `steer` ist (die Standardeinstellung, keine Konfiguration erforderlich), versucht OpenClaw, diese Eingabeaufforderung an die aktive Laufzeit zu senden. OpenClaw und das native Codex-App-Server-Harness implementieren die Details der Zustellung unterschiedlich.

Diese Seite behandelt die Steuerung über den Warteschlangenmodus für normale eingehende Nachrichten im Modus `steer`. Im Modus `followup` oder `collect` überspringen normale Nachrichten diesen Pfad und warten, bis der aktive Durchlauf beendet ist. Informationen zum expliziten Befehl `/steer <message>` finden Sie unter [Steuern](/de/tools/steer).

## Laufzeitgrenze

Die Steuerung unterbricht keinen bereits laufenden Tool-Aufruf. OpenClaw prüft an Modellgrenzen auf Nachrichten in der Steuerungswarteschlange:

1. Der Assistent fordert Tool-Aufrufe an.
2. OpenClaw führt den Tool-Aufruf-Batch der aktuellen Assistentennachricht aus.
3. OpenClaw gibt das Ereignis zum Ende des Turns aus.
4. OpenClaw leert die Warteschlange mit Steuerungsnachrichten.
5. OpenClaw fügt diese Nachrichten vor dem nächsten LLM-Aufruf als Benutzernachrichten an.

Dadurch bleiben Tool-Ergebnisse mit der Assistentennachricht verknüpft, von der sie angefordert wurden, und der nächste Modellaufruf erhält anschließend die neuesten Benutzereingaben.

Das native Codex-App-Server-Harness stellt `turn/steer` anstelle der internen Steuerungswarteschlange der OpenClaw-Laufzeit bereit. OpenClaw sammelt Eingabeaufforderungen in der Warteschlange während des konfigurierten Ruhefensters und sendet anschließend eine einzelne `turn/steer`-Anfrage mit allen gesammelten Benutzereingaben in der Reihenfolge ihres Eingangs.

Codex-Review- und manuelle Compaction-Turns lehnen eine Steuerung innerhalb desselben Turns ab. Wenn eine Laufzeit im Modus `steer` keine Steuerung akzeptieren kann, wartet OpenClaw, bis der aktive Durchlauf beendet ist, bevor die Eingabeaufforderung gestartet wird.

## Modi

| Modus        | Verhalten bei aktivem Durchlauf                                    | Späteres Verhalten                                                                      |
| ----------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `steer`     | Steuert die Eingabeaufforderung nach Möglichkeit in die aktive Laufzeit. | Wartet auf das Ende des aktiven Durchlaufs, wenn keine Steuerung verfügbar ist.                      |
| `followup`  | Führt keine Steuerung durch.                                        | Führt Nachrichten aus der Warteschlange später nach dem Ende des aktiven Durchlaufs aus.                               |
| `collect`   | Führt keine Steuerung durch.                                        | Fasst kompatible Nachrichten aus der Warteschlange nach dem Entprellfenster zu einem späteren Turn zusammen. |
| `interrupt` | Bricht den aktiven Durchlauf ab, statt ihn zu steuern.          | Startet nach dem Abbruch die neueste Nachricht.                                           |

## Beispiel für einen Nachrichtenstoß

Wenn vier Benutzer Nachrichten senden, während der Agent einen Tool-Aufruf ausführt:

- Beim Standardverhalten empfängt die aktive Laufzeit alle vier Nachrichten vor ihrer nächsten Modellentscheidung in der Reihenfolge ihres Eingangs. OpenClaw entnimmt sie an der nächsten Modellgrenze aus der Warteschlange; Codex empfängt sie als einen gebündelten `turn/steer`.
- Mit `/queue collect` führt OpenClaw keine Steuerung durch. Es wartet, bis der aktive Durchlauf beendet ist, und erstellt dann nach dem Entprellfenster einen Folgeturn mit kompatiblen Nachrichten aus der Warteschlange.
- Mit `/queue interrupt` bricht OpenClaw den aktiven Durchlauf ab und startet die neueste Nachricht, statt eine Steuerung durchzuführen.

## Geltungsbereich

Die Steuerung richtet sich immer an den aktuellen aktiven Sitzungsdurchlauf. Sie erstellt keine neue Sitzung, ändert nicht die Tool-Richtlinie des aktiven Durchlaufs und trennt Nachrichten nicht nach Absender. In Kanälen mit mehreren Benutzern enthalten eingehende Eingabeaufforderungen bereits Absender- und Routingkontext, sodass der nächste Modellaufruf erkennen kann, wer die jeweilige Nachricht gesendet hat.

Verwenden Sie `followup` oder `collect`, wenn Nachrichten standardmäßig in die Warteschlange gestellt werden sollen, anstatt den aktiven Durchlauf zu steuern. Verwenden Sie `interrupt`, wenn die neueste Eingabeaufforderung den aktiven Durchlauf ersetzen soll.

## Entprellung

Die integrierte Warteschlangenentprellung gilt für die Zustellung von `followup` und `collect` aus der Warteschlange. Im Modus `steer` mit dem nativen Codex-Harness legt sie außerdem das Ruhefenster fest, bevor gebündelte `turn/steer` gesendet werden. Bei OpenClaw verwendet die aktive Steuerung selbst keinen Entprell-Timer, da OpenClaw Nachrichten auf natürliche Weise bis zur nächsten Modellgrenze bündelt.

## Verwandte Themen

- [Befehlswarteschlange](/de/concepts/queue)
- [Steuern](/de/tools/steer)
- [Nachrichten](/de/concepts/messages)
- [Agentenschleife](/de/concepts/agent-loop)
