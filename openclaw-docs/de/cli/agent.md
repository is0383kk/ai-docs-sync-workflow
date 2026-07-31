---
read_when:
    - Sie möchten einen Agent-Durchlauf über Skripte ausführen (und die Antwort optional zustellen)
summary: CLI-Referenz für `openclaw agent` (einen Agent-Turn über das Gateway senden)
title: Agent
x-i18n:
    generated_at: "2026-07-26T18:51:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1a4c139a3b235d6a56ba63063737b80f93448c2dbb7a92c6d0756fb19a9f95e4
    source_path: cli/agent.md
    workflow: 16
---

# `openclaw agent`

Führen Sie einen Agent-Durchlauf über das Gateway aus. Das explizite Flag `--local` ist der einzige eingebettete Ausführungspfad.

Übergeben Sie mindestens einen Sitzungsselektor: `--to`, `--session-key`, `--session-id` oder `--agent`.

Verwandt: [Tool zum Senden an Agents](/de/tools/agent-send)

## Optionen

- `-m, --message <text>`: Nachrichtentext
- `--message-file <path>`: Nachrichtentext aus einer UTF-8-Datei lesen
- `-t, --to <dest>`: Empfänger, aus dem der Sitzungsschlüssel abgeleitet wird
- `--session-key <key>`: expliziter Sitzungsschlüssel für das Routing
- `--session-id <id>`: explizite Sitzungs-ID
- `--agent <id>`: Agent-ID; überschreibt Routing-Zuordnungen
- `--model <id>`: Modellüberschreibung für diesen Durchlauf (`provider/model` oder Modell-ID)
- `--thinking <level>`: Denkstufe des Agents (`off`, `minimal`, `low`, `medium`, `high` sowie vom Provider unterstützte benutzerdefinierte Stufen wie `xhigh`, `adaptive` oder `max`)
- `--verbose <on|off>`: Ausführlichkeitsstufe für die Sitzung beibehalten
- `--channel <channel>`: Zustellungskanal; weglassen, um den Hauptkanal der Sitzung zu verwenden
- `--reply-to <target>`: Überschreibung des Zustellungsziels
- `--reply-channel <channel>`: Überschreibung des Zustellungskanals
- `--reply-account <id>`: Überschreibung des Zustellungskontos
- `--local`: eingebetteten Agent direkt ausführen (nach dem Vorladen der Plugin-Registry)
- `--deliver`: Antwort an den ausgewählten Kanal bzw. das ausgewählte Ziel zurücksenden
- `--timeout <seconds>`: Frist für den Agent-Durchlauf dieses Befehls überschreiben (Standardwert 600 oder `agents.defaults.timeoutSeconds`); `0` deaktiviert die Gesamtfrist. Der Ausweichwert von 600 Sekunden gehört zu diesem CLI-Befehl, nicht zu gewöhnlichen Gateway-Durchläufen, deren Standardwert 48 Stunden beträgt.
- `--json`: JSON ausgeben

## Beispiele

```bash
openclaw agent --to +15555550123 --message "status update" --deliver
openclaw agent --agent ops --message "Summarize logs"
openclaw agent --agent ops --message-file ./task.md
openclaw agent --agent ops --model openai/gpt-5.4 --message "Summarize logs"
openclaw agent --session-key agent:ops:incident-42 --message "Summarize status"
openclaw agent --agent ops --session-key incident-42 --message "Summarize status"
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json
openclaw agent --agent ops --message "Generate report" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "Run locally" --local
```

## Hinweise

- Übergeben Sie genau eine der Optionen `--message` oder `--message-file`. `--message-file` entfernt eine vorangestellte UTF-8-BOM und behält mehrzeilige Inhalte bei; Dateien ohne gültige UTF-8-Codierung werden abgelehnt. Dateien mit mehr als 4 MiB werden vor der Weiterleitung abgelehnt.
- Slash-Befehle (zum Beispiel `/compact`) können nicht über `--message` ausgeführt werden. Die CLI lehnt sie ab und verweist stattdessen auf den dedizierten Befehl (`openclaw sessions compact <key>` für Compaction).
- Durchläufe mit `--local` sind einmalig: Gebündelte MCP-Loopback-Ressourcen und für den Durchlauf geöffnete betriebsbereite Claude-stdio-Sitzungen werden nach der Antwort beendet, sodass skriptgesteuerte Aufrufe keine lokalen Kindprozesse weiterlaufen lassen. Gateway-gestützte Durchläufe behalten Gateway-eigene MCP-Loopback-Ressourcen stattdessen im laufenden Gateway-Prozess.
- Die eigenständige eingebettete Ausführung mit `--local` verweigert die Wiederverwendung einer bestehenden Hauptsitzung, solange die Wiederherstellung nach einem Neustart aussteht. Führen Sie den Durchlauf über ein funktionsfähiges Gateway aus oder setzen Sie die Sitzung dort mit `/new` oder `/reset` zurück; ein unabhängiger eingebetteter Prozess kann die Zuständigkeit für diese Wiederherstellung nicht sicher mit dem Gateway-Scanner koordinieren.
- Wenn `--agent`, `--channel` und `--to` gemeinsam verwendet werden, folgt das Sitzungs-Routing dem kanonischen Empfänger des Kanals und `session.dmScope`. Kanäle mit einer stabilen, ausschließlich ausgehenden Empfängeridentität verwenden eine Provider-eigene Sitzung, die von der Hauptsitzung des Agents isoliert ist. `--reply-channel` und `--reply-account` wirken sich nur auf die Zustellung aus.
- `--session-key` wählt einen expliziten Sitzungsschlüssel aus. Mit einem Agent-Präfix versehene Schlüssel müssen `agent:<agent-id>:<session-key>` verwenden, und `--agent` muss mit der Agent-ID des Schlüssels übereinstimmen, wenn beide angegeben sind. Reine Schlüssel, die keine Sentinel-Schlüssel sind, werden bei Angabe von `--agent` darauf beschränkt, andernfalls auf den konfigurierten Standard-Agent; beispielsweise wird `--agent ops --session-key incident-42` an `agent:ops:incident-42` weitergeleitet. Die literalen Schlüssel `global` und `unknown` bleiben nur dann ohne Gültigkeitsbereich, wenn kein `--agent` angegeben ist.
- `--json` reserviert stdout für die JSON-Antwort; Diagnosemeldungen von Gateway, Plugin und `--local` werden an stderr ausgegeben, sodass Skripte stdout direkt analysieren können.
- Nachdem die Wiederholungsversuche für vorübergehende Handshake-Fehler ausgeschöpft sind, führt ein Gateway-Timeout oder eine geschlossene Verbindung zum Fehlschlagen des Befehls; die CLI führt den Durchlauf niemals stillschweigend eingebettet erneut aus. Ein Transportverlust ist mehrdeutig – das Gateway hat den Durchlauf möglicherweise angenommen und schließt ihn eventuell noch ab. Daher weist der Hinweis auf stderr darauf hin, vor einem erneuten Versuch oder einer erneuten Ausführung mit `--local` zuerst `openclaw gateway status` und das Sitzungsprotokoll zu prüfen, um eine doppelte Ausführung des Durchlaufs zu vermeiden.
- `SIGTERM`/`SIGINT` unterbrechen eine wartende Gateway-gestützte Anfrage; wenn das Gateway den Durchlauf bereits angenommen hat, sendet die CLI vor dem Beenden außerdem `chat.abort` für diese Durchlauf-ID. Durchläufe mit `--local` empfangen dasselbe Signal, senden jedoch kein `chat.abort`. Ein Kindprozess des Launchers, der durch das erste weitergeleitete `SIGINT` bzw. `SIGTERM` beendet wird, endet mit Status 130 bzw. 143. Wenn für den internen Schlüssel zur Deduplizierung von Durchläufen bereits ein aktiver Durchlauf für diese Sitzung existiert, meldet die Antwort `status: "in_flight"`, und die CLI ohne JSON-Ausgabe gibt statt einer leeren Antwort eine Diagnosemeldung auf stderr aus. Behalten Sie für externe Cron-/systemd-Wrapper eine Absicherung zum erzwungenen Beenden wie `timeout -k 60 600 openclaw agent ...` bei, damit der Supervisor den Prozess beseitigen kann, falls das Herunterfahren nicht ordnungsgemäß abgeschlossen werden kann.
- Wenn dieser Befehl die Neugenerierung von `models.json` auslöst, werden durch SecretRef verwaltete Provider-Anmeldedaten als nicht geheime Markierungen beibehalten (zum Beispiel Namen von Umgebungsvariablen, `secretref-env:ENV_VAR_NAME` oder `secretref-managed`), niemals als aufgelöster geheimer Klartext. Die geschriebenen Markierungen stammen aus dem aktiven Snapshot der Quellkonfiguration, nicht aus aufgelösten geheimen Laufzeitwerten.

## JSON-Zustellungsstatus

Mit `--json --deliver` enthält die JSON-Antwort der CLI das oberste Feld `deliveryStatus`, damit Skripte zwischen zugestellten, unterdrückten, teilweise zugestellten und fehlgeschlagenen Sendungen unterscheiden können:

```json
{
  "payloads": [{ "text": "Report ready", "mediaUrl": null }],
  "meta": { "durationMs": 1200 },
  "deliveryStatus": {
    "requested": true,
    "attempted": true,
    "status": "sent",
    "succeeded": true,
    "resultCount": 1
  }
}
```

Gateway-gestützte CLI-Antworten behalten außerdem die unverarbeitete Ergebnisstruktur des Gateways unter `result.deliveryStatus` bei.

`deliveryStatus.status` ist einer der folgenden Werte:

| Status           | Bedeutung                                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `sent`           | Zustellung abgeschlossen.                                                                                                                        |
| `suppressed`     | Die Zustellung wurde absichtlich nicht gesendet (zum Beispiel weil ein Hook zum Senden von Nachrichten sie abgebrochen hat oder kein sichtbares Ergebnis vorlag). Endgültig, kein erneuter Versuch. |
| `partial_failed` | Mindestens eine Nutzlast wurde gesendet, bevor eine spätere Nutzlast fehlschlug.                                                                                   |
| `failed`         | Es wurde keine dauerhafte Sendung abgeschlossen oder die Vorabprüfung der Zustellung ist fehlgeschlagen.                                                                                   |

Allgemeine Felder:

- `requested`: immer `true`, wenn das Objekt vorhanden ist.
- `attempted`: `true`, sobald der dauerhafte Sendepfad ausgeführt wurde; `false` bei Fehlern der Vorabprüfung oder wenn keine sichtbaren Nutzlasten vorhanden sind.
- `succeeded`: `true`, `false` oder `"partial"`; `"partial"` wird mit `status: "partial_failed"` kombiniert.
- `reason`: Grund in kleingeschriebener Snake-Case-Schreibweise aus der dauerhaften Zustellung oder Vorabvalidierung. Zu den bekannten Werten gehören `cancelled_by_message_sending_hook`, `no_visible_payload`, `no_visible_result`, `channel_resolved_to_internal`, `unknown_channel`, `invalid_delivery_target` und `no_delivery_target`; fehlgeschlagene dauerhafte Sendungen können außerdem die fehlgeschlagene Phase melden. Behandeln Sie unbekannte Werte als undurchsichtig, da die Menge erweitert werden kann.
- `resultCount`: Anzahl der Ergebnisse von Kanalsendungen, sofern verfügbar.
- `sentBeforeError`: `true`, wenn bei einem teilweisen Fehlschlag mindestens eine Nutzlast gesendet wurde, bevor ein Fehler auftrat.
- `error`: `true` bei fehlgeschlagenen oder teilweise fehlgeschlagenen Sendungen.
- `errorMessage`: nur vorhanden, wenn die Meldung eines zugrunde liegenden Zustellungsfehlers erfasst wurde. Fehler der Vorabprüfung enthalten `error`/`reason`, aber kein `errorMessage`.
- `payloadOutcomes`: optionale Ergebnisse pro Nutzlast mit `index`, `status`, `reason`, `resultCount`, `error`, `stage`, `sentBeforeError` oder Hook-Metadaten, sofern verfügbar.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Agent-Laufzeit](/de/concepts/agent)
