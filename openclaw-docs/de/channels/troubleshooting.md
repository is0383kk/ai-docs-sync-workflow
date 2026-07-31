---
read_when:
    - Der Kanaltransport meldet eine bestehende Verbindung, aber Antworten schlagen fehl
    - Vor der ausführlichen Provider-Dokumentation sind kanalspezifische Prüfungen erforderlich
summary: Schnelle Fehlerbehebung auf Kanalebene mit kanalspezifischen Fehlersignaturen und Lösungen
title: Fehlerbehebung für Kanäle
x-i18n:
    generated_at: "2026-07-26T18:50:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3891595e4b5aca9de7997a6e908fa1c9246579032bfdfa1656a6992d644c3ecc
    source_path: channels/troubleshooting.md
    workflow: 16
---

Verwenden Sie diese Seite, wenn ein Kanal eine Verbindung herstellt, das Verhalten aber fehlerhaft ist.

## Befehlsabfolge

Führen Sie zunächst diese Befehle der Reihe nach aus:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Fehlerfreier Ausgangszustand:

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`, `write-capable` oder `admin-capable`
- Die Kanalprüfung zeigt, dass der Transport verbunden ist und, sofern unterstützt, `works` oder `audit ok`

## Nach einer Aktualisierung

Verwenden Sie dies, wenn Telegram, iMessage, Konfigurationen aus der BlueBubbles-Ära oder ein anderer Plugin-Kanal
nach der Aktualisierung verschwindet.

```bash
openclaw status --all
openclaw doctor --fix
openclaw gateway restart
openclaw status --all
```

Suchen Sie in `openclaw
status --all` nach `plugin load failed: dependency tree corrupted; run openclaw doctor --fix`. Dies bedeutet, dass der Kanal konfiguriert ist, die Einrichtung bzw. das Laden des Plugins jedoch auf einen beschädigten
Abhängigkeitsbaum gestoßen ist, statt den Kanal zu registrieren. `openclaw doctor --fix` entfernt veraltete
Abhängigkeitssymlinks der Plugin-Laufzeit und veraltete Authentifizierungsschatten; anschließend lädt `openclaw gateway restart`
einen bereinigten Zustand neu.

## WhatsApp

### WhatsApp-Fehlersignaturen

| Symptom                             | Schnellste Prüfung                                       | Behebung                                                                                                                              |
| ----------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Verbunden, aber keine Antworten auf Direktnachrichten         | `openclaw pairing list whatsapp`                    | Genehmigen Sie den Absender oder ändern Sie die Direktnachrichtenrichtlinie bzw. Zulassungsliste.                                                                                    |
| Gruppennachrichten werden ignoriert              | Prüfen Sie `requireMention` und die Erwähnungsmuster in der Konfiguration | Erwähnen Sie den Bot oder lockern Sie die Erwähnungsrichtlinie für diese Gruppe.                                                                          |
| QR-Anmeldung läuft mit 408 ab         | Prüfen Sie die Gateway-Umgebungsvariablen `HTTPS_PROXY` / `HTTP_PROXY`      | Legen Sie einen erreichbaren Proxy fest; verwenden Sie `NO_PROXY` nur für Umgehungen.                                                                         |
| Zufällige Verbindungsabbrüche/Neuanmeldeschleifen     | `openclaw channels status --probe` und Protokolle           | Kürzliche Neuverbindungen werden auch bei aktuell bestehender Verbindung markiert; beobachten Sie die Protokolle, starten Sie das Gateway neu und verknüpfen Sie das Konto erneut, falls die Verbindung weiterhin instabil ist. |
| `status=408 Request Time-out`-Schleife  | Prüfung, Protokolle, Doctor und anschließend Gateway-Status            | Beheben Sie zuerst die Konnektivitäts- bzw. Zeitsteuerungsprobleme des Hosts; sichern Sie die Authentifizierungsdaten und verknüpfen Sie das Konto erneut, falls die Schleife weiterhin besteht.                                   |
| Antworten treffen Sekunden/Minuten verspätet ein | `openclaw doctor --fix`                             | Doctor beendet nachweislich veraltete lokale TUI-Clients, wenn diese die Gateway-Ereignisschleife beeinträchtigen.                                    |

Vollständige Fehlerbehebung: [WhatsApp-Fehlerbehebung](/de/channels/whatsapp#troubleshooting)

## Telegram

### Telegram-Fehlersignaturen

| Symptom                              | Schnellste Prüfung                                    | Behebung                                                                                                                    |
| ------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `/start`, aber kein nutzbarer Antwortablauf    | `openclaw pairing list telegram`                 | Genehmigen Sie die Kopplung oder ändern Sie die Direktnachrichtenrichtlinie.                                                                                   |
| Bot ist online, aber die Gruppe bleibt stumm    | Prüfen Sie die Anforderung für Erwähnungen und den Datenschutzmodus des Bots  | Deaktivieren Sie den Datenschutzmodus für die Sichtbarkeit in Gruppen oder erwähnen Sie den Bot.                                                              |
| Sendefehler mit Netzwerkfehlern    | Prüfen Sie die Protokolle auf fehlgeschlagene Telegram-API-Aufrufe      | Korrigieren Sie das DNS-/IPv6-/Proxy-Routing zu `api.telegram.org`.                                                                      |
| Beim Start wird `getMe returned 401` gemeldet | Prüfen Sie die konfigurierte Tokenquelle                    | Kopieren Sie das BotFather-Token erneut oder generieren Sie es neu und aktualisieren Sie `botToken`, `tokenFile` oder das `TELEGRAM_BOT_TOKEN` des Standardkontos. |
| Polling bleibt hängen oder stellt die Verbindung langsam wieder her  | `openclaw logs --follow` für die Polling-Diagnose | Führen Sie ein Upgrade durch; anhaltende Blockaden weisen normalerweise auf Proxy-/DNS-/IPv6-Probleme hin.                                                            |
| `setMyCommands` wird beim Start abgelehnt  | Prüfen Sie die Protokolle auf `BOT_COMMANDS_TOO_MUCH`         | Reduzieren Sie benutzerdefinierte Telegram-Befehle bzw. Befehle von Plugins und Skills oder deaktivieren Sie native Menüs.                                                  |
| Nach dem Upgrade werden Sie von der Zulassungsliste blockiert    | `openclaw security audit` und die Konfigurations-Zulassungslisten  | Führen Sie `openclaw doctor --fix` aus oder ersetzen Sie `@username` durch numerische Absender-IDs.                                            |

Vollständige Fehlerbehebung: [Telegram-Fehlerbehebung](/de/channels/telegram#troubleshooting)

## Discord

### Discord-Fehlersignaturen

| Symptom                                   | Schnellste Prüfung                                                                                                                | Behebung                                                                                                                                                                                                                                                                   |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bot ist online, antwortet aber nicht in der Guild           | `openclaw channels status --probe`                                                                                           | Erlauben Sie die Guild bzw. den Kanal und prüfen Sie den Intent für Nachrichteninhalte.                                                                                                                                                                                                                |
| Gruppennachrichten werden ignoriert                    | Prüfen Sie die Protokolle auf durch Erwähnungsregeln verworfene Nachrichten                                                                                          | Erwähnen Sie den Bot oder legen Sie für die Guild bzw. den Kanal `requireMention: false` fest.                                                                                                                                                                                                             |
| Tipp-/Token-Nutzung, aber keine Discord-Nachricht | Prüfen Sie, ob es sich um ein Ereignis in einem Ambient-Raum oder einen teilnehmenden `message_tool`-Raum handelt, in dem das Modell `message(action=send)` ausgelassen hat | Prüfen Sie das ausführliche Gateway-Protokoll auf Metadaten unterdrückter endgültiger Nutzlasten, überprüfen Sie `messages.groupChat.unmentionedInbound`, lesen Sie [Ereignisse in Ambient-Räumen](/de/channels/ambient-room-events) oder behalten Sie `messages.groupChat.visibleReplies: "automatic"` für normale Gruppenanfragen bei. |
| Antworten auf Direktnachrichten fehlen                        | `openclaw pairing list discord`                                                                                              | Genehmigen Sie die Kopplung für Direktnachrichten oder passen Sie die Direktnachrichtenrichtlinie an.                                                                                                                                                                                                                               |

Vollständige Fehlerbehebung: [Discord-Fehlerbehebung](/de/channels/discord#troubleshooting)

## Slack

### Slack-Fehlersignaturen

| Symptom                                | Schnellste Prüfung                             | Behebung                                                                                                                                                  |
| -------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Socket-Modus verbunden, aber keine Antworten | `openclaw channels status --probe`        | Überprüfen Sie das App-Token, das Bot-Token und die erforderlichen Berechtigungsbereiche; achten Sie bei SecretRef-basierten Einrichtungen auf `botTokenStatus` / `appTokenStatus = configured_unavailable`. |
| Direktnachrichten werden blockiert                            | `openclaw pairing list slack`             | Genehmigen Sie die Kopplung oder lockern Sie die Direktnachrichtenrichtlinie.                                                                                                                  |
| Kanalnachricht wird ignoriert                | Prüfen Sie `groupPolicy` und die Kanal-Zulassungsliste | Erlauben Sie den Kanal oder ändern Sie die Richtlinie in `open`.                                                                                                        |

Vollständige Fehlerbehebung: [Slack-Fehlerbehebung](/de/channels/slack#troubleshooting)

## iMessage

### iMessage-Fehlersignaturen

| Symptom                              | Schnellste Prüfung                                           | Behebung                                                                   |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------- |
| `imsg` fehlt oder schlägt auf anderen Systemen als macOS fehl | `openclaw channels status --probe --channel imessage`   | Führen Sie OpenClaw auf dem Messages-Mac aus oder verwenden Sie einen SSH-Wrapper für `cliPath`. |
| Senden ist unter macOS möglich, Empfangen jedoch nicht     | Prüfen Sie die macOS-Datenschutzberechtigungen für die Messages-Automatisierung | Erteilen Sie die TCC-Berechtigungen erneut und starten Sie den Kanalprozess neu.                 |
| Absender von Direktnachrichten wird blockiert                    | `openclaw pairing list imessage`                        | Genehmigen Sie die Kopplung oder aktualisieren Sie die Zulassungsliste.                                  |

Vollständige Fehlerbehebung: [iMessage-Fehlerbehebung](/de/channels/imessage#troubleshooting)

## Signal

### Signal-Fehlersignaturen

| Symptom                         | Schnellste Prüfung                              | Behebung                                                      |
| ------------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| Daemon erreichbar, aber Bot reagiert nicht | `openclaw channels status --probe`         | Überprüfen Sie die Daemon-URL bzw. das Konto für `signal-cli` und den Empfangsmodus. |
| Direktnachricht wird blockiert                      | `openclaw pairing list signal`             | Genehmigen Sie den Absender oder passen Sie die Direktnachrichtenrichtlinie an.                      |
| Gruppenantworten werden nicht ausgelöst    | Prüfen Sie die Gruppen-Zulassungsliste und die Erwähnungsmuster | Fügen Sie den Absender bzw. die Gruppe hinzu oder lockern Sie die Zugriffsregeln.                       |

Vollständige Fehlerbehebung: [Signal-Fehlerbehebung](/de/channels/signal#troubleshooting)

## QQ Bot

### QQ-Bot-Fehlersignaturen

| Symptom                         | Schnellste Prüfung                               | Behebung                                                             |
| ------------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| Bot antwortet „zum Mars verschwunden“      | Überprüfen Sie `appId` und `clientSecret` in der Konfiguration | Legen Sie die Anmeldedaten fest oder starten Sie das Gateway neu.                         |
| Keine eingehenden Nachrichten             | `openclaw channels status --probe`          | Überprüfen Sie die Anmeldedaten auf der QQ Open Platform.                     |
| Sprache wird nicht transkribiert           | Prüfen Sie die Konfiguration des STT-Providers                   | Konfigurieren Sie `channels.qqbot.stt` oder `tools.media.audio`.          |
| Proaktive Nachrichten kommen nicht an | Prüfen Sie die Interaktionsanforderungen der QQ-Plattform  | QQ blockiert möglicherweise vom Bot initiierte Nachrichten, wenn kürzlich keine Interaktion stattgefunden hat. |

Vollständige Fehlerbehebung: [QQ-Bot-Fehlerbehebung](/de/channels/qqbot#troubleshooting)

## Matrix

### Matrix-Fehlersignaturen

| Symptom                             | Schnellste Prüfung                          | Behebung                                                                       |
| ----------------------------------- | -------------------------------------- | ------------------------------------------------------------------------- |
| Angemeldet, aber Raumnachrichten werden ignoriert | `openclaw channels status --probe`     | Prüfen Sie `groupPolicy`, die Raum-Zulassungsliste und die Erwähnungsbeschränkung.                  |
| Direktnachrichten werden nicht verarbeitet                  | `openclaw pairing list matrix`         | Genehmigen Sie den Absender oder passen Sie die Richtlinie für Direktnachrichten an.                                       |
| Verschlüsselte Räume funktionieren nicht                | `openclaw matrix verify status`        | Verifizieren Sie das Gerät erneut und prüfen Sie anschließend `openclaw matrix verify backup status`.  |
| Die Wiederherstellung der Sicherung steht aus oder ist fehlgeschlagen    | `openclaw matrix verify backup status` | Führen Sie `openclaw matrix verify backup restore` aus oder wiederholen Sie den Vorgang mit einem Wiederherstellungsschlüssel. |
| Cross-Signing/Bootstrap scheint fehlerhaft | `openclaw matrix verify bootstrap`     | Reparieren Sie den geheimen Speicher, das Cross-Signing und den Sicherungsstatus in einem Durchgang.       |

Vollständige Einrichtung und Konfiguration: [Matrix](/de/channels/matrix)

## Verwandte Themen

- [Kopplung](/de/channels/pairing)
- [Kanalrouting](/de/channels/channel-routing)
- [Gateway-Fehlerbehebung](/de/gateway/troubleshooting)
