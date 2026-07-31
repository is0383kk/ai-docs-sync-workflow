---
read_when:
    - Sie möchten ein Systemereignis in die Warteschlange einreihen, ohne einen Cron-Job zu erstellen
    - Sie müssen Heartbeats aktivieren oder deaktivieren
    - Sie möchten die Systemanwesenheitseinträge überprüfen
summary: CLI-Referenz für `openclaw system` (Systemereignisse, Heartbeat, Präsenz)
title: System
x-i18n:
    generated_at: "2026-07-26T18:23:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aaca206d8b463fd33f9e3cb21382bbf36469e9daa2706d8a9e2c7fab14b76e7a
    source_path: cli/system.md
    workflow: 16
---

# `openclaw system`

Hilfsfunktionen auf Systemebene für das Gateway: Systemereignisse in die Warteschlange einreihen, Heartbeats steuern und Präsenz anzeigen.

Alle `system`-Unterbefehle verwenden Gateway-RPC und akzeptieren die gemeinsamen Client-Flags:

| Flag              | Standardwert                              | Beschreibung                                                                                                                                                                                            |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--url <url>`     | `gateway.remote.url`, wenn konfiguriert | Gateway-WebSocket-URL.                                                                                                                                                                                 |
| `--token <token>` | keiner                                 | Gateway-Token (falls erforderlich).                                                                                                                                                                           |
| `--timeout <ms>`  | `30000`                              | RPC-Zeitüberschreitung in Millisekunden.                                                                                                                                                                           |
| `--expect-final`  | aus                                  | Auf die endgültige Antwort warten (Agent).                                                                                                                                                                       |
| `--json`          | aus                                  | JSON ausgeben. `heartbeat last/enable/disable` und `system presence` geben unabhängig von diesem Flag immer die unverarbeitete JSON-Nutzlast des RPC aus; `system event` verwendet es, um zwischen JSON und einer einfachen `ok`-Zeile umzuschalten. |

## Häufig verwendete Befehle

```bash
openclaw system event --text "Auf dringende Folgeaufgaben prüfen" --mode now
openclaw system event --text "Auf dringende Folgeaufgaben prüfen" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

Reiht standardmäßig ein Systemereignis in der **Hauptsitzung** ein. Der nächste Heartbeat fügt es als `System:`-Zeile in den Prompt ein. Verwenden Sie `--mode now`, um den Heartbeat sofort auszulösen; `next-heartbeat` (Standardwert) wartet bis zum nächsten geplanten Durchlauf.

Übergeben Sie `--session-key`, um eine bestimmte Sitzung als Ziel festzulegen, beispielsweise um den Abschluss einer asynchronen Aufgabe an den Kanal zurückzumelden, der sie gestartet hat.

<Note>
**Zeitliche Ausnahme bei `--session-key`:** Wenn `--session-key` angegeben wird, wird `--mode next-heartbeat` zu einem sofortigen gezielten Weckvorgang, anstatt auf den nächsten geplanten Durchlauf zu warten. Gezielte Weckvorgänge verwenden die Heartbeat-Absicht `immediate` und umgehen dadurch die Sperre des Runners für noch nicht fällige Ausführungen, die andernfalls einen Weckvorgang mit der Absicht `event` aufschieben (und effektiv verwerfen) würde. Wenn Sie eine verzögerte Zustellung wünschen, lassen Sie `--session-key` weg, damit das Ereignis in der Hauptsitzung landet und mit dem nächsten regulären Heartbeat verarbeitet wird.
</Note>

Flags:

- `--text <text>`: erforderlicher Text des Systemereignisses.
- `--mode <mode>`: `now` oder `next-heartbeat` (Standardwert).
- `--session-key <sessionKey>`: optional; legt eine bestimmte Agent-Sitzung als Ziel fest, anstatt der Hauptsitzung des Agenten. Schlüssel, die nicht zum ermittelten Agenten gehören, greifen auf die Hauptsitzung des Agenten zurück.

## `system heartbeat last|enable|disable`

- `last`: zeigt das letzte Heartbeat-Ereignis an.
- `enable`: aktiviert Heartbeats wieder (verwenden Sie dies, wenn sie deaktiviert wurden).
- `disable`: pausiert Heartbeats.

## `system presence`

Listet die aktuellen Systempräsenzeinträge auf, die dem Gateway bekannt sind (Nodes, Instanzen und ähnliche Statuszeilen).

## Hinweise

- Erfordert ein laufendes Gateway, das über Ihre aktuelle Konfiguration erreichbar ist (lokal oder remote).
- Systemereignisse sind flüchtig und bleiben über Neustarts hinweg nicht erhalten.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
