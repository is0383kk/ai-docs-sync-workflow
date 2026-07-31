---
read_when:
    - Arbeiten an Funktionen des Nextcloud-Talk-Kanals
summary: Supportstatus, Funktionen und Konfiguration von Nextcloud Talk
title: Nextcloud Talk
x-i18n:
    generated_at: "2026-07-26T17:39:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59f4fe51555bcb13d630140866307b1a49ba077059818ec116ee50ef0c877b2b
    source_path: channels/nextcloud-talk.md
    workflow: 16
---

Nextcloud Talk ist ein herunterladbares Kanal-Plugin (`@openclaw/nextcloud-talk`), das OpenClaw über einen Talk-Webhook-Bot mit einer selbst gehosteten Nextcloud-Instanz verbindet. Direktnachrichten, Räume, Reaktionen und Markdown-Nachrichten werden unterstützt; Medien werden als URLs versendet.

## Installation

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

Verwenden Sie die reine Paketspezifikation, um dem aktuellen offiziellen Release-Tag zu folgen. Fixieren Sie eine exakte Version nur, wenn Sie eine reproduzierbare Installation benötigen.

Aus einem lokalen Checkout (Entwicklungsabläufe):

```bash
openclaw plugins install ./path/to/local/nextcloud-talk-plugin
```

Starten Sie das Gateway nach der Installation neu. Details: [Plugins](/de/tools/plugin)

## Schnelleinrichtung (für Einsteiger)

1. Installieren Sie das Plugin (siehe oben).
2. Erstellen Sie auf Ihrem Nextcloud-Server einen Bot:

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature webhook --feature response --feature reaction
   ```

   Behalten Sie `--feature response` bei: Ohne diese Funktion schlagen ausgehende Antworten mit 401 fehl. Reparieren Sie einen vorhandenen Bot mit `./occ talk:bot:state --feature webhook --feature response --feature reaction <botId> 1`.

3. Aktivieren Sie den Bot in den Einstellungen des Zielraums.
4. Konfigurieren Sie OpenClaw:
   - Konfiguration: `channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - Oder Umgebungsvariablen: `NEXTCLOUD_TALK_BOT_SECRET` (nur Standardkonto)

   CLI-Einrichtung (`--url`/`--token` sind Aliasse für die expliziten Felder; `nc-talk` und `nc` funktionieren als Kanal-Aliasse):

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --url https://cloud.example.com \
     --token "<shared-secret>"
   ```

   Entsprechende explizite Felder:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret "<shared-secret>"
   ```

   Dateibasierter geheimer Schlüssel:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret-file /path/to/nextcloud-talk-secret
   ```

5. Starten Sie das Gateway neu (oder schließen Sie die Einrichtung ab).

Minimale Konfiguration:

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## Hinweise

- Bots können keine Direktnachrichten initiieren. Der Benutzer muss dem Bot zuerst eine Nachricht senden.
- Die Webhook-URL muss vom Nextcloud-Server aus erreichbar sein; legen Sie `webhookPublicUrl` fest, wenn sich das Gateway hinter einem Proxy befindet. Webhook-Anfragen werden mit dem geheimen Schlüssel des Bots per HMAC-SHA256 signiert; ungültige Signaturen werden abgelehnt und unterliegen einer Ratenbegrenzung.
- Medien-Uploads werden von der Bot-API nicht unterstützt; ausgehende Medien werden als `Attachment: <url>`-Zeile angehängt.
- Die Webhook-Nutzlast unterscheidet nicht zwischen Direktnachrichten und Räumen; legen Sie `apiUser` + `apiPassword` fest, um Abfragen des Raumtyps zu aktivieren (etwa 5 Minuten zwischengespeichert). Ohne diese Angaben wird jede Unterhaltung als Raum behandelt.
- Ausgehende Anfragen durchlaufen den SSRF-Schutz. Für einen Nextcloud-Host in einem vertrauenswürdigen privaten/internen Netzwerk können Sie dies mit `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork: true` ausdrücklich zulassen.
- Wenn `apiUser`/`apiPassword` und `webhookPublicUrl` festgelegt sind, prüft `openclaw channels status` den Bot und warnt, wenn die Funktion `response` fehlt.

## Zugriffskontrolle (Direktnachrichten)

- Standard: `channels.nextcloud-talk.dmPolicy = "pairing"`. Unbekannte Absender erhalten einen Kopplungscode.
- Genehmigen über:
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- Öffentliche Direktnachrichten: `channels.nextcloud-talk.dmPolicy="open"` plus `channels.nextcloud-talk.allowFrom=["*"]`.
- `allowFrom` gleicht ausschließlich Nextcloud-Benutzer-IDs ab (in Kleinbuchstaben); Anzeigenamen werden ignoriert.

## Räume (Gruppen)

- Standard: `channels.nextcloud-talk.groupPolicy = "allowlist"` (Erwähnung erforderlich).
- Setzen Sie Räume mit `channels.nextcloud-talk.rooms` auf die Zulassungsliste, wobei der Raum-Token als Schlüssel dient; `"*"` legt einen Platzhalterstandard fest:

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- Raumspezifische Schlüssel: `requireMention` (standardmäßig true), `enabled` (false deaktiviert den Raum), `allowFrom` (raumspezifische Absender-Zulassungsliste), `tools` (Zulassen/Ablehnen-Überschreibungen für Tools), `skills` (geladene Skills begrenzen), `systemPrompt`.
- Um keine Räume zuzulassen, lassen Sie die Zulassungsliste leer oder legen Sie `channels.nextcloud-talk.groupPolicy="disabled"` fest.

## Funktionen

| Funktion        | Status              |
| --------------- | ------------------- |
| Direktnachrichten | Unterstützt       |
| Räume           | Unterstützt         |
| Threads         | Nicht unterstützt   |
| Medien          | Nur URLs            |
| Reaktionen      | Unterstützt         |
| Native Befehle  | Nicht unterstützt   |

## Konfigurationsreferenz (Nextcloud Talk)

Vollständige Konfiguration: [Konfiguration](/de/gateway/configuration)

Provider-Optionen:

- `channels.nextcloud-talk.enabled`: Start des Kanals aktivieren/deaktivieren.
- `channels.nextcloud-talk.baseUrl`: URL der Nextcloud-Instanz.
- `channels.nextcloud-talk.botSecret`: gemeinsamer geheimer Schlüssel des Bots (Zeichenfolge oder Geheimnisreferenz).
- `channels.nextcloud-talk.botSecretFile`: Pfad zum geheimen Schlüssel in einer regulären Datei. Symbolische Verknüpfungen werden abgelehnt.
- `channels.nextcloud-talk.apiUser`: API-Benutzer für Raumabfragen (Erkennung von Direktnachrichten) und die Statusprüfung.
- `channels.nextcloud-talk.apiPassword`: API-/App-Passwort für Raumabfragen.
- `channels.nextcloud-talk.apiPasswordFile`: Pfad zur API-Passwortdatei.
- `channels.nextcloud-talk.webhookPort`: Port des Webhook-Listeners (Standard: 8788).
- `channels.nextcloud-talk.webhookHost`: Webhook-Host (Standard: 0.0.0.0).
- `channels.nextcloud-talk.webhookPath`: Webhook-Pfad (Standard: /nextcloud-talk-webhook).
- `channels.nextcloud-talk.webhookPublicUrl`: extern erreichbare Webhook-URL.
- `channels.nextcloud-talk.dmPolicy`: `pairing | allowlist | open | disabled` (Standard: pairing). `open` erfordert `allowFrom=["*"]`.
- `channels.nextcloud-talk.allowFrom`: Zulassungsliste für Direktnachrichten (Benutzer-IDs).
- `channels.nextcloud-talk.groupPolicy`: `allowlist | open | disabled` (Standard: allowlist).
- `channels.nextcloud-talk.groupAllowFrom`: Absender-Zulassungsliste für Räume (Benutzer-IDs); verwendet `allowFrom` als Rückfall, wenn nicht festgelegt.
- `channels.nextcloud-talk.rooms`: raumspezifische Einstellungen und Zulassungsliste (siehe oben).
- Statische Absender-Zugriffsgruppen können aus `allowFrom` und `groupAllowFrom` mit `accessGroup:<name>` referenziert werden.
- `channels.nextcloud-talk.historyLimit`: Verlaufslimit für Gruppen (0 deaktiviert).
- `channels.nextcloud-talk.dmHistoryLimit`: Verlaufslimit für Direktnachrichten (0 deaktiviert).
- `channels.nextcloud-talk.dms`: nach Benutzer-ID indizierte Überschreibungen pro Direktnachricht (`historyLimit`).
- `channels.nextcloud-talk.textChunkLimit`: Segmentgröße für ausgehenden Text in Zeichen (Standard: 4000).
- `channels.nextcloud-talk.streaming.chunkMode`: `length` (Standard) oder `newline`, um vor der längenbasierten Segmentierung an Leerzeilen (Absatzgrenzen) zu trennen.
- `channels.nextcloud-talk.streaming.block.enabled`: Block-Streaming für diesen Kanal aktivieren oder deaktivieren.
- `channels.nextcloud-talk.streaming.block.coalesce`: Abstimmung der Zusammenführung beim Block-Streaming.
- `channels.nextcloud-talk.responsePrefix`: Präfix für ausgehende Antworten.
- `channels.nextcloud-talk.markdown.tables`: Darstellungsmodus für Markdown-Tabellen (`off | bullets | code | block`).
- `channels.nextcloud-talk.mediaMaxMb`: Größenbegrenzung für eingehende Medien (MB).
- `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork`: private/interne Nextcloud-Hosts am SSRF-Schutz vorbei zulassen.
- `channels.nextcloud-talk.accounts.<id>`: kontospezifische Überschreibungen (dieselben Schlüssel); `defaultAccount` wählt das Standardkonto aus. Die Umgebungsvariablen `NEXTCLOUD_TALK_BOT_SECRET` / `NEXTCLOUD_TALK_API_PASSWORD` gelten nur für das Standardkonto.

## Verwandte Themen

- [Kanalübersicht](/de/channels) — alle unterstützten Kanäle
- [Kopplung](/de/channels/pairing) — Authentifizierung von Direktnachrichten und Kopplungsablauf
- [Gruppen](/de/channels/groups) — Verhalten von Gruppenchats und Erwähnungspflicht
- [Kanal-Routing](/de/channels/channel-routing) — Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Härtung
