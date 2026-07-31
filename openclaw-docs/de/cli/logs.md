---
read_when:
    - Sie müssen Gateway-Protokolle remote verfolgen (ohne SSH)
    - Sie möchten JSON-Protokollzeilen für Werkzeuge
summary: CLI-Referenz für `openclaw logs` (Gateway-Protokolle per RPC fortlaufend anzeigen)
title: Protokolle
x-i18n:
    generated_at: "2026-07-26T17:42:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

Gateway-Dateiprotokolle über RPC fortlaufend ausgeben. Funktioniert im Remote-Modus.

## Optionen

- `--limit <n>`: maximale Anzahl zurückzugebender Protokollzeilen (Standard: `200`)
- `--max-bytes <n>`: maximale Anzahl aus der Protokolldatei zu lesender Bytes (Standard: `250000`)
- `--follow`: dem Protokollstream folgen
- `--interval <ms>`: Abfrageintervall beim Folgen (Standard: `1000`)
- `--json`: zeilengetrennte JSON-Ereignisse ausgeben
- `--plain`: reine Textausgabe ohne formatierte Darstellung
- `--no-color`: ANSI-Farben deaktivieren
- `--local-time`: Zeitstempel in Ihrer lokalen Zeitzone darstellen (Standard)
- `--utc`: Zeitstempel in UTC darstellen

## Gemeinsame Gateway-RPC-Optionen

- `--url <url>`: Gateway-WebSocket-URL
- `--token <token>`: Gateway-Token
- `--timeout <ms>`: Zeitüberschreitung in ms (Standard: `30000`)
- `--expect-final`: auf eine abschließende Antwort warten, wenn der Gateway-Aufruf von einem Agenten ausgeführt wird

Die Übergabe von `--url` überspringt automatisch angewendete Anmeldedaten aus der Konfiguration; geben Sie `--token` ausdrücklich an, wenn das Ziel-Gateway eine Authentifizierung erfordert.

## Beispiele

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

Das ausgewählte Stammprofil entspricht der rotierenden Datei des Gateways: Das Standardprofil
verwendet `openclaw-YYYY-MM-DD.log`, während benannte Profile
`openclaw-<profile>-YYYY-MM-DD.log` verwenden (zum Beispiel
`openclaw-dev-YYYY-MM-DD.log`).

## Fallback- und Wiederherstellungsverhalten

- Wenn das implizite lokale Loopback-Gateway eine Kopplung anfordert, die Verbindung während des Aufbaus schließt oder eine Zeitüberschreitung eintritt, bevor `logs.tail` antwortet, greift `openclaw logs` automatisch auf die konfigurierte Gateway-Protokolldatei zurück. Explizite `--url`-Ziele verwenden diesen Fallback niemals.
- `--follow` greift nach einem RPC-Fehler des impliziten lokalen Gateways nicht auf diese konfigurierte Datei zurück – eine veraltete, parallel vorhandene Datei könnte eine aktive fortlaufende Ausgabe verfälschen. Unter Linux wird stattdessen, sofern verfügbar, anhand der PID das aktive Gateway-Journal des Benutzer-systemd verwendet (die ausgewählte Quelle wird ausgegeben); andernfalls wird die Verbindung zum aktiven Gateway weiterhin erneut versucht.
- Während `--follow` lösen vorübergehende Verbindungsabbrüche (Schließen des WebSockets, Zeitüberschreitung, Verbindungsabbruch) eine automatische Wiederverbindung mit exponentiellem Backoff aus: bis zu 8 Wiederholungsversuche, mit höchstens 30s zwischen den Versuchen. Bei jedem Wiederholungsversuch wird eine Warnung an stderr ausgegeben, und sobald eine Abfrage erfolgreich ist, wird einmalig ein `[logs] gateway reconnected`-Hinweis ausgegeben. Im `--json`-Modus werden beide als `{"type":"notice"}`-Datensätze an stderr ausgegeben. Nicht behebbare Fehler (Authentifizierungsfehler, fehlerhafte Konfiguration) führen weiterhin zum sofortigen Beenden.
- Im `--follow --json`-Modus werden Wechsel der Protokollquelle als `{"type":"meta"}`-Datensätze ausgegeben. Verfolgen Sie Cursor für jede `sourceKind` separat: Ein Stream kann von der Ausgabe der Gateway-Datei (`sourceKind: "file"`) zum Fallback auf das lokale Journal (`sourceKind: "journal"`, `localFallback: true`, mit `service.pid`/`service.unit`) und nach der Wiederherstellung zurück zur Ausgabe der Gateway-Datei wechseln. Gehen Sie nicht davon aus, dass während der gesamten Sitzung eine stabile Quelle oder ein stabiler Cursor verwendet wird, und tolerieren Sie überlappende Zeilen, wenn bei der Wiederherstellung der Cursor der Gateway-Datei erneut abgespielt wird.

## Verwandte Themen

- [Protokollierungsübersicht](/de/logging)
- [Gateway-CLI](/de/cli/gateway)
- [CLI-Referenz](/de/cli)
- [Gateway-Protokollierung](/de/gateway/logging)
