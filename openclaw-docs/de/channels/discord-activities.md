---
read_when:
    - Discord-Aktivitätswidgets einrichten oder Fehler beheben
summary: Eigenständige OpenClaw-HTML-Widgets in Discord Activities starten
title: Discord-Aktivitäten
x-i18n:
    generated_at: "2026-07-26T18:14:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b1bc04443aef89fd514290c3bebdbdd3e9972298b45cae3806bec99344f6d8cd
    source_path: channels/discord-activities.md
    workflow: 16
---

Discord Activities ermöglichen es einem Agenten, ein interaktives, eigenständiges HTML-Widget im aktuellen Discord-Kanal zu veröffentlichen. Die Nachricht enthält die Schaltfläche **Open widget**; durch Anklicken wird das Widget innerhalb von Discord geöffnet.

Die Funktion ist standardmäßig deaktiviert. OpenClaw registriert die Activity-HTTP-Routen, das Agentenwerkzeug `show_widget` und den Handler für die Startschaltfläche nur, wenn `channels.discord.activities` vorhanden ist und ein Client-Secret aufgelöst werden kann. Der veraltete Alias `discord_widget` bleibt für eine Version verfügbar.

## Voraussetzungen

- ein vorhandener [OpenClaw-Discord-Bot](/de/channels/discord)
- ein öffentlicher HTTPS-Hostname, über den das OpenClaw-Gateway erreichbar ist
- die Berechtigung, Activities und OAuth2 für die Discord-Anwendung des Bots zu konfigurieren

Jeder HTTPS-Reverse-Proxy oder Tunnel ist geeignet. Ein benannter Cloudflare Tunnel stellt einen stabilen Hostnamen bereit, ohne den Gateway-Port direkt offenzulegen.

```yaml
# ~/.cloudflared/config.yml
tunnel: openclaw-discord
credentials-file: /home/you/.cloudflared/TUNNEL-ID.json
ingress:
  - hostname: openclaw.example.com
    service: http://127.0.0.1:18789
  - service: http_status:404
```

```bash
cloudflared tunnel login
cloudflared tunnel create openclaw-discord
cloudflared tunnel route dns openclaw-discord openclaw.example.com
cloudflared tunnel run openclaw-discord
```

Lassen Sie die normale Gateway-Authentifizierung aktiviert. Nur das Activity-Präfix ist öffentlich; das Plugin validiert OAuth, die Mitgliedschaft in der Activity-Instanz, die Kanalbindung, Sitzungen und einmalig verwendbare Dokumentberechtigungen selbst.

## Einrichtung

<Steps>
  <Step title="Gateway über HTTPS verfügbar machen">
    Starten Sie Ihren Tunnel oder Reverse-Proxy und überprüfen Sie nach dem Hinzufügen der Activities-Konfiguration, ob `https://openclaw.example.com/discord/activity/` das Gateway erreicht. Ersetzen Sie den Beispiel-Hostnamen durch Ihren eigenen.
  </Step>

  <Step title="Activities in Discord aktivieren">
    Öffnen Sie die vorhandene Bot-Anwendung im [Discord Developer Portal](https://discord.com/developers/applications). Öffnen Sie **Activities**, aktivieren Sie Activities und erstellen Sie eine URL-Zuordnung:

    - Präfix: `ROOT` (`/`)
    - Ziel: `openclaw.example.com/discord/activity`

    Das Ziel besteht aus dem öffentlichen Hostnamen und `/discord/activity`, ohne abschließenden Schrägstrich.

  </Step>

  <Step title="OAuth2-Client-Secret kopieren">
    Öffnen Sie **OAuth2** im Developer Portal. Discord erfordert mindestens einen Weiterleitungs-URI. Fügen Sie daher einen lokalen Platzhalter wie die Loopback-Adresse hinzu, falls für die Anwendung noch keiner vorhanden ist; das Embedded App SDK übernimmt den Activity-Rückgabeablauf. Kopieren Sie das Client-Secret der Anwendung oder setzen Sie es zurück. Behandeln Sie es als Zugangsdaten: Fügen Sie es weder in Chats oder Protokolle noch in eine eingecheckte Konfigurationsdatei ein.
  </Step>

  <Step title="OpenClaw konfigurieren">
    Fügen Sie dem Discord-Konto, das Widgets anbieten soll, einen Block hinzu:

    ```json5
    {
      channels: {
        discord: {
          token: "${DISCORD_BOT_TOKEN}",
          activities: {
            clientSecret: "${DISCORD_CLIENT_SECRET}",
            // Optional. Standardmäßig wird die beim Start ermittelte Bot-Anwendungs-ID verwendet.
            applicationId: "YOUR_DISCORD_APPLICATION_ID",
          },
        },
      },
    }
    ```

    Sie können `clientSecret` aus dem Block weglassen, wenn `DISCORD_CLIENT_SECRET` festgelegt ist. Der Block selbst muss vorhanden bleiben, damit die Funktion aktiviert wird.

    Die normalen Discord-Zugriffseinstellungen bleiben davon unabhängig. Beispielsweise steuert `allowFrom` weiterhin, wer dem Agenten Direktnachrichten senden darf; die Einstellung steuert nicht, wer ein bereits in einem Kanal veröffentlichtes Widget öffnen darf.

  </Step>

  <Step title="Neu starten und testen">
    Starten Sie das Gateway neu. Bitten Sie den Agenten in einer Discord-Unterhaltung, ein interaktives Widget anzuzeigen. Der Agent ruft `show_widget` auf; klicken Sie in der veröffentlichten Nachricht auf **Open widget**.
  </Step>
</Steps>

## Sicherheitsmodell

- OAuth identifiziert den Discord-Benutzer, bevor Widget-Metadaten zurückgegeben werden.
- Die Get Activity Instance API von Discord muss bestätigen, dass der OAuth-Benutzer in der aktuellen Activity-Instanz anwesend ist. Der Kanal der Instanz muss mit dem Kanal übereinstimmen, in dem das Widget veröffentlicht wurde.
- Jeder, dem Discord den Zugriff auf diesen Kanal erlaubt, kann dessen Widgets öffnen. Um den Personenkreis einzuschränken, verwenden Sie die Discord-Kanalberechtigungen. OpenClaw-Befehls- und Direktnachrichten-Zulassungslisten gewähren oder entziehen keinen Zugriff auf bereits veröffentlichte Kanalinhalte.
- OAuth-Sitzungen laufen nach 15 Minuten ab. Widget-Dokumentberechtigungen laufen nach 60 Sekunden ab und können einmal verwendet werden.
- Widgets laufen nach sieben Tagen ab, wobei höchstens 64 pro Discord-Plugin-Instanz aufbewahrt werden.
- Das Widget-HTML wird von Ihrem Agenten erstellt und sollte als vertrauenswürdiger Inhalt behandelt werden. Betten Sie keine Geheimnisse ein, die ein fehlerhaftes Widget nicht offenlegen soll.
- Das Widget kann innerhalb seines eigenen verschachtelten Frames navigieren. Der `sandbox="allow-scripts"`-iframe blockiert Navigation auf oberster Ebene, Pop-ups und Same-Origin-Zugriff, während seine Content Security Policy Netzwerkverbindungen und externe Ressourcen blockiert. Diese Kontrollen dienen der mehrschichtigen Absicherung und bilden keine Sicherheitsgrenze gegenüber dem Agenten, der das Widget erstellt hat.
- Wenn Activities deaktiviert ist, wird `/discord/activity` überhaupt nicht registriert.

Die öffentliche Activity-Shell und die Route für den Token-Austausch werden nach der Aktivierung über Ihren Tunnel erreichbar. Ohne eine gültige OAuth-Sitzung und eine einmalig verwendbare Dokumentberechtigung legen sie kein Widget-HTML offen.

## Fehlerbehebung

### Die Activity meldet „Gateway offline“

- Stellen Sie sicher, dass der Tunnel ausgeführt wird und zum tatsächlich gebundenen Port des Gateways weiterleitet.
- Stellen Sie sicher, dass das Ziel im Developer Portal `/discord/activity` enthält.
- Starten Sie das Gateway nach Änderungen an der Discord- oder OpenClaw-Konfiguration neu.
- Prüfen Sie die Gateway-Protokolle auf die einzeilige Warnung zu einem fehlenden Activities-Client-Secret.

### Discord öffnet eine leere Seite oder meldet `blocked:csp`

- Überprüfen Sie, ob die URL-Zuordnung `ROOT` verwendet und kein zweites `/discord/activity`-Segment hinzufügt.
- Stellen Sie sicher, dass die Shell, `shell.js` und das SDK-Modul vollständig über den Discord-Proxy zurückgegeben werden.
- Untersuchen Sie die Gateway-Protokolle auf Anfragen unter `/discord/activity/`.

Netzwerkanfragen von Widgets werden absichtlich blockiert. Betten Sie alle vom Widget benötigten CSS-, JavaScript-, Bild- und Dateninhalte direkt ein.

### „Widget unavailable“

Öffnen Sie die Startschaltfläche in dem Kanal, in dem der Agent sie veröffentlicht hat. OpenClaw erfasst Starts beim Anklicken serverseitig. Daher kann ein neuer Starteintrag das genaue Widget ermitteln, selbst wenn Discord die benutzerdefinierte ID der Schaltfläche auslässt oder beschädigt. Wenn weder die benutzerdefinierte ID noch ein Starteintrag aufgelöst werden kann, öffnet OpenClaw das zuletzt in diesem Kanal veröffentlichte, noch aktive Widget. Ältere Widgets bleiben über Schaltflächen erreichbar, die ihre benutzerdefinierte ID beibehalten.

### „You cannot launch Activities in this channel“

Discord startet Activities nicht aus Threads von Forumsbeiträgen. OpenClaw kann dort die Widget-Nachricht und die Schaltfläche veröffentlichen, aber starten Sie die Activity stattdessen aus einem regulären Textkanal. Diese Einschränkung stammt von Discord, nicht von OpenClaw.
