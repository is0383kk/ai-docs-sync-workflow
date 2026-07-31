---
read_when:
    - Diagnose der Channel-Konnektivität oder des Gateway-Zustands
    - Grundlegendes zu CLI-Befehlen und -Optionen für Zustandsprüfungen
summary: Befehle für Integritätsprüfungen und Überwachung des Gateway-Status
title: Systemdiagnosen
x-i18n:
    generated_at: "2026-07-26T18:23:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59a7fbfb7fb86be7dbd3a03f96c7328c2bc8cc851230c0bdd1b1b750b3014be4
    source_path: gateway/health.md
    workflow: 16
---

Kurzanleitung zur Überprüfung der Kanalkonnektivität ohne Mutmaßungen.

## Schnellprüfungen

- `openclaw status` – lokale Zusammenfassung: Gateway-Erreichbarkeit/-Modus, Aktualisierungshinweis, Alter der verknüpften Kanalauthentifizierung, Sitzungen und letzte Aktivität.
- `openclaw status --all` – vollständige lokale Diagnose (schreibgeschützt, farbig, kann zur Fehlerbehebung sicher eingefügt werden).
- `openclaw status --deep` – fordert das laufende Gateway zu einer Live-Prüfung auf (`health` mit `probe:true`), einschließlich Kanalprüfungen pro Konto, sofern unterstützt.
- `openclaw status --usage` – zeigt Momentaufnahmen der Nutzung und des Kontingents des Modell-Providers.
- `openclaw health` – fordert vom laufenden Gateway dessen Zustandsmomentaufnahme an (nur WS; keine direkten Kanalsockets von der CLI).
- `openclaw health --verbose` (Alias `--debug`) – erzwingt eine Live-Zustandsprüfung und gibt Details zur Gateway-Verbindung aus.
- `openclaw health --json` – maschinenlesbare Ausgabe der Zustandsmomentaufnahme.
- Senden Sie `/status` als eigenständigen Chatbefehl in einem beliebigen Kanal, um eine Statusantwort zu erhalten, ohne den Agenten aufzurufen.
- Protokolle: Führen Sie `openclaw logs --follow` (oder `openclaw --profile <profile> logs --follow`) aus und filtern Sie nach `web-heartbeat`, `web-reconnect`, `web-auto-reply`, `web-inbound`.

Bei Discord und anderen Chat-Providern geben Sitzungszeilen keine Auskunft über die Socket-Verfügbarkeit.
`openclaw sessions`, Gateway `sessions.list` und das Agentenwerkzeug `sessions_list`
lesen den gespeicherten Konversationszustand. Ein Provider kann die Verbindung wiederherstellen und einen fehlerfreien Kanalstatus
anzeigen, bevor eine neue Sitzungszeile angelegt wurde. Verwenden Sie die oben genannten Kanalstatus- und
Zustandsbefehle für Live-Konnektivitätsprüfungen.

## Detaillierte Diagnose

- Anmeldedaten auf dem Datenträger: `ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (die Änderungszeit sollte aktuell sein).
- Sitzungsspeicher: `ls -l ~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. Anzahl und letzte Empfänger werden über `status` angezeigt.
- Erneute Verknüpfung: `openclaw channels logout && openclaw channels login --verbose`, wenn die Statuscodes 409-515 oder `loggedOut` in den Protokollen erscheinen. Der QR-Anmeldeablauf wird nach der Kopplung bei Status 515 automatisch einmal neu gestartet.
- Diagnosen sind standardmäßig aktiviert (`diagnostics.enabled: false` deaktiviert sie). Speicherereignisse erfassen RSS-/Heap-Bytezahlen sowie Schwellenwert- und Wachstumsdruck. Verfügbarkeitswarnungen erfassen Ereignisschleifenverzögerung/-auslastung, das Verhältnis der CPU-Kerne sowie die Anzahl aktiver, wartender und in der Warteschlange befindlicher Sitzungen, wenn der Prozess ausgeführt wird, aber ausgelastet ist. Ereignisse zu übergroßen Nutzlasten erfassen, was abgelehnt, gekürzt oder in Blöcke aufgeteilt wurde, einschließlich Größen und Grenzwerten, jedoch niemals Nachrichtentext, Anhangsinhalte, Webhook-Inhalte, unformatierte Anfrage-/Antwortinhalte, Tokens, Cookies oder geheime Werte.
- Derselbe Heartbeat steuert den begrenzten Stabilitätsrekorder: `openclaw gateway stability` (oder den `diagnostics.stability`-Gateway-RPC). Bei schwerwiegenden Gateway-Beendigungen, Zeitüberschreitungen beim Herunterfahren und Startfehlern nach einem Neustart wird die neueste Momentaufnahme unter `~/.openclaw/logs/stability/` gespeichert. Prüfen Sie das neueste Paket mit `openclaw gateway stability --bundle latest`.
- Führen Sie für Fehlerberichte `openclaw gateway diagnostics export` aus und hängen Sie die generierte ZIP-Datei an: eine Markdown-Zusammenfassung, das neueste Stabilitätspaket, bereinigte Protokollmetadaten, bereinigte Gateway-Status-/Zustandsmomentaufnahmen und die Konfigurationsstruktur. Chattext, Webhook-Inhalte, Werkzeugausgaben, Anmeldedaten, Cookies, Konto-/Nachrichtenkennungen und geheime Werte werden ausgelassen oder unkenntlich gemacht. Siehe [Diagnoseexport](/de/gateway/diagnostics).

## Konfiguration der Zustandsüberwachung

- `channels.<provider>.healthMonitor.enabled`: Deaktiviert Neustarts durch die Zustandsüberwachung für einen bestimmten Kanal, während die globale Überwachung aktiviert bleibt.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: Überschreibung für mehrere Konten, die Vorrang vor der Einstellung auf Kanalebene hat.
- Diese kanalbezogenen Überschreibungen gelten für die integrierten Kanäle, die sie derzeit bereitstellen: Discord, Google Chat, iMessage, IRC, Microsoft Teams, Signal, Slack, Telegram und WhatsApp.

## Verfügbarkeitsüberwachung

Externe Dienste zur Verfügbarkeitsüberwachung sollten den dedizierten Endpunkt `/health` verwenden, nicht `/v1/chat/completions`.

- **Verwenden:** `GET /health` – sofortige Antwort, keine Sitzung wird erstellt, kein LLM-Aufruf, gibt `{"ok":true,"status":"live"}` zurück
- **Nicht verwenden:** `/v1/chat/completions` für Zustandsprüfungen – jede Anfrage erstellt eine vollständige Agentensitzung mit Skills-Momentaufnahme, Kontextzusammenstellung und LLM-Aufrufen

Wenn weder der Header `x-openclaw-session-key` noch das Feld `user` angegeben ist, generiert `/v1/chat/completions` für jede Anfrage eine neue zufällige Sitzung. Überwachungsdienste, die alle 15 Minuten eine Anfrage senden, erstellen etwa 96 Sitzungen/Tag, von denen jede 4-22KB belegt. Dies führt mit der Zeit zu einem aufgeblähten Sitzungsspeicher und kann eine Überschreitung des Kontextfensters verursachen.

### Einrichtungsbeispiele für Überwachungsdienste

- **BetterStack:** Legen Sie die URL für die Zustandsprüfung auf `https://<your-gateway-host>:<port>/health` fest
- **UptimeRobot:** Fügen Sie einen neuen HTTP-Monitor mit der URL `https://<your-gateway-host>:<port>/health` hinzu
- **Allgemein:** Jede HTTP-GET-Anfrage an `/health` gibt 200 mit `{"ok":true}` zurück, wenn das Gateway fehlerfrei ist

## Wenn etwas fehlschlägt

- `logged out` oder Status 409-515 → Verknüpfung mit `openclaw channels logout` und anschließend `openclaw channels login` erneut herstellen.
- Gateway nicht erreichbar → starten Sie es: `openclaw gateway --port 18789` (verwenden Sie `--force`, wenn der Port belegt ist).
- Keine eingehenden Nachrichten → bestätigen Sie, dass das verknüpfte Telefon online und der Absender zugelassen ist (`channels.whatsapp.allowFrom`); stellen Sie bei Gruppenchats sicher, dass Zulassungsliste und Erwähnungsregeln übereinstimmen (`channels.whatsapp.groups`, `agents.entries.*.groupChat.mentionPatterns`).

## Dedizierter „health“-Befehl

`openclaw health` fordert vom laufenden Gateway dessen Zustandsmomentaufnahme an (keine direkten
Kanalsockets von der CLI). Standardmäßig gibt der Befehl eine aktuelle zwischengespeicherte Gateway-Momentaufnahme zurück, während das
Gateway diesen Cache im Hintergrund aktualisiert; `--verbose` erzwingt stattdessen eine Live-Prüfung.
Der Befehl meldet, sofern verfügbar, das Alter der verknüpften Anmeldedaten/Authentifizierung, Prüfungszusammenfassungen pro Kanal,
eine Zusammenfassung des Sitzungsspeichers und die Prüfungsdauer. Er wird mit einem von null verschiedenen Status beendet, wenn das Gateway
nicht erreichbar ist oder die Prüfung fehlschlägt bzw. eine Zeitüberschreitung auftritt.

Optionen:

- `--json`: maschinenlesbare JSON-Ausgabe
- `--timeout <ms>`: überschreibt die standardmäßige Prüfungszeitüberschreitung von 10s
- `--verbose`: erzwingt eine Live-Prüfung und gibt Details zur Gateway-Verbindung aus
- `--debug`: Alias für `--verbose`

Die Zustandsmomentaufnahme enthält: `ok` (boolescher Wert), `ts` (Zeitstempel), `durationMs` (Prüfungszeit), Status pro Kanal, Agentenverfügbarkeit und eine Zusammenfassung des Sitzungsspeichers.

## Verwandte Themen

- [Gateway-Betriebshandbuch](/de/gateway)
- [Diagnoseexport](/de/gateway/diagnostics)
- [Gateway-Fehlerbehebung](/de/gateway/troubleshooting)
