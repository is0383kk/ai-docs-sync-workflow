---
read_when:
    - Arbeiten an Funktionen des Google-Chat-Kanals
summary: Status, Funktionen und Konfiguration der Google Chat-App-Unterstützung
title: Google Chat
x-i18n:
    generated_at: "2026-07-26T18:48:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d3fb96564294b57040327bb21ab7331bf8412eb04f879a9c7ea1018ba2bddab
    source_path: channels/googlechat.md
    workflow: 16
---

Google Chat wird als offizielles `@openclaw/googlechat`-Plugin ausgeführt: Direktnachrichten und Bereiche über Webhooks der Google Chat API (nur HTTP-Endpunkt, kein Pub/Sub).

## Installation

```bash
openclaw plugins install @openclaw/googlechat
```

Lokaler Checkout (bei Ausführung aus einem Git-Repository):

```bash
openclaw plugins install ./path/to/local/googlechat-plugin
```

## Schnelleinrichtung (Einsteiger)

1. Erstellen Sie ein Google-Cloud-Projekt und aktivieren Sie die **Google Chat API**.
   - Rufen Sie folgende Seite auf: [Google Chat API Credentials](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - Aktivieren Sie die API, falls sie noch nicht aktiviert ist.
2. Erstellen Sie ein **Service Account**:
   - Klicken Sie auf **Create Credentials** > **Service Account**.
   - Geben Sie einen beliebigen Namen ein (z. B. `openclaw-chat`).
   - Lassen Sie Berechtigungen und Hauptkonten leer (**Continue**, dann **Done**).
3. Erstellen Sie den **JSON-Schlüssel** und laden Sie ihn herunter:
   - Klicken Sie auf das neue Dienstkonto > Registerkarte **Keys** > **Add Key** > **Create new key** > **JSON** > **Create**.
4. Speichern Sie die heruntergeladene JSON-Datei auf Ihrem Gateway-Host (z. B. `~/.openclaw/googlechat-service-account.json`).
5. Erstellen Sie unter [Google Cloud Console Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat) eine Google-Chat-App:
   - Füllen Sie **Application info** aus (App-Name, Avatar-URL, Beschreibung).
   - Aktivieren Sie **Interactive features**.
   - Aktivieren Sie unter **Functionality** die Option **Join spaces and group conversations**.
   - Wählen Sie unter **Connection settings** die Option **HTTP endpoint URL** aus.
   - Wählen Sie unter **Triggers** die Option **Use a common HTTP endpoint URL for all triggers** aus und legen Sie als Wert Ihre öffentliche Gateway-URL gefolgt von `/googlechat` fest (siehe [Öffentliche URL](#public-url-webhook-only)).
   - Aktivieren Sie unter **Visibility** die Option **Make this Chat app available to specific people and groups in `<Your Domain>`** und geben Sie Ihre E-Mail-Adresse ein.
   - Klicken Sie auf **Save**.
6. Aktivieren Sie den App-Status: Aktualisieren Sie die Seite, suchen Sie **App status**, legen Sie ihn auf **Live - available to users** fest und klicken Sie erneut auf **Save**.
7. Konfigurieren Sie OpenClaw mit dem Dienstkonto und der Webhook-Zielgruppe (muss mit der Konfiguration der Chat-App übereinstimmen):
   - Umgebungsvariable: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json` (nur Standardkonto) oder
   - Konfiguration: siehe [Wichtige Konfigurationsoptionen](#config-highlights). `openclaw channels add --channel googlechat` akzeptiert außerdem `--audience-type`, `--audience`, `--webhook-path` und `--webhook-url`.
8. Starten Sie das Gateway. Google Chat sendet POST-Anfragen an Ihren Webhook-Pfad (standardmäßig `/googlechat`).

## Zu Google Chat hinzufügen

Sobald das Gateway ausgeführt wird und Ihre E-Mail-Adresse in der Sichtbarkeitsliste enthalten ist:

1. Rufen Sie [Google Chat](https://chat.google.com/) auf.
2. Klicken Sie neben **Direct Messages** auf das Symbol **+** (Plus).
3. Suchen Sie nach dem **App name**, den Sie in der Google Cloud Console konfiguriert haben.
   - Der Bot wird _nicht_ in der Marketplace-Übersicht angezeigt, da es sich um eine private App handelt; suchen Sie anhand seines Namens nach ihm.
4. Wählen Sie den Bot aus, klicken Sie auf **Add** oder **Chat** und senden Sie eine Nachricht.

## Öffentliche URL (nur Webhook)

Google-Chat-Webhooks benötigen einen öffentlichen HTTPS-Endpunkt. Geben Sie aus Sicherheitsgründen **nur den Pfad `/googlechat`** für das Internet frei und halten Sie das OpenClaw-Dashboard sowie andere Endpunkte privat.

### Option A: Tailscale Funnel (empfohlen)

Verwenden Sie Tailscale Serve für das private Dashboard und Funnel für den öffentlichen Webhook-Pfad.

1. Prüfen Sie, an welche Adresse Ihr Gateway gebunden ist:

   ```bash
   ss -tlnp | grep 18789
   ```

   Notieren Sie die IP-Adresse (z. B. `127.0.0.1`, `0.0.0.0` oder eine Tailscale-Adresse vom Typ `100.x.x.x`).

2. Geben Sie das Dashboard ausschließlich für das Tailnet frei (Port 8443):

   ```bash
   # Bei Bindung an localhost (127.0.0.1 oder 0.0.0.0):
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # Bei ausschließlicher Bindung an eine Tailscale-IP:
   tailscale serve --bg --https 8443 http://100.x.x.x:18789
   ```

3. Geben Sie ausschließlich den Webhook-Pfad öffentlich frei:

   ```bash
   # Bei Bindung an localhost (127.0.0.1 oder 0.0.0.0):
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # Bei ausschließlicher Bindung an eine Tailscale-IP:
   tailscale funnel --bg --set-path /googlechat http://100.x.x.x:18789/googlechat
   ```

4. Wenn Sie dazu aufgefordert werden, rufen Sie die in der Ausgabe angezeigte Autorisierungs-URL auf, um Funnel für diese Node zu aktivieren.

5. Überprüfen Sie die Konfiguration:

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

Ihre öffentliche Webhook-URL lautet `https://<node-name>.<tailnet>.ts.net/googlechat`; das Dashboard bleibt unter `https://<node-name>.<tailnet>.ts.net:8443/` ausschließlich über das Tailnet erreichbar. Verwenden Sie die öffentliche URL (ohne `:8443`) in der Konfiguration der Google-Chat-App.

> Hinweis: Diese Konfiguration bleibt über Neustarts hinweg bestehen. Entfernen Sie sie später mit `tailscale funnel reset` und `tailscale serve reset`.

### Option B: Reverse-Proxy (Caddy)

Leiten Sie ausschließlich den Webhook-Pfad über den Proxy weiter:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

Anfragen an `your-domain.com/` werden ignoriert oder mit 404 beantwortet, während `your-domain.com/googlechat` an OpenClaw weitergeleitet wird.

### Option C: Cloudflare Tunnel

Konfigurieren Sie die Ingress-Regeln des Tunnels so, dass nur der Webhook-Pfad weitergeleitet wird:

- **Path**: `/googlechat` -> `http://localhost:18789/googlechat`
- **Default rule**: HTTP 404 (Not Found)

## Funktionsweise

1. Google Chat sendet JSON per POST an den Webhook-Pfad des Gateways (nur POST, JSON-Inhaltstyp erforderlich, Rate-Limit pro IP).
2. OpenClaw authentifiziert jede Anfrage vor der Weiterleitung:
   - Chat-App-Ereignisse enthalten `Authorization: Bearer <token>`; das Token wird überprüft, bevor der vollständige Body geparst wird.
   - Ereignisse von Google Workspace Add-ons enthalten das Token im Body (`authorizationEventObject.systemIdToken`) und werden vor der Überprüfung innerhalb eines strengeren Budgets vor der Authentifizierung gelesen (16 KB, 3 s).
3. Das Token wird anhand von `audienceType` + `audience` geprüft:
   - `audienceType: "app-url"` → Die Zielgruppe ist Ihre HTTPS-Webhook-URL.
   - `audienceType: "project-number"` → Die Zielgruppe ist die Cloud-Projektnummer.
   - Add-on-Tokens unter `app-url` erfordern zusätzlich, dass `appPrincipal` auf die numerische OAuth-2.0-Client-ID der App gesetzt ist (21 Ziffern, keine E-Mail-Adresse); andernfalls schlägt die Überprüfung fehl und eine Warnung wird protokolliert.
4. Nachrichten werden anhand des Bereichs weitergeleitet:
   - Bereiche erhalten bereichsspezifische Sitzungen `agent:<agentId>:googlechat:group:<spaceId>`; Antworten werden an den Nachrichten-Thread gesendet.
   - Direktnachrichten werden standardmäßig in der Hauptsitzung des Agenten zusammengeführt; legen Sie `session.dmScope` für kontaktbezogene Direktnachrichten-Sitzungen fest (siehe [Sitzung](/de/concepts/session)).
5. Der Zugriff per Direktnachricht erfolgt standardmäßig über Kopplung. Unbekannte Absender erhalten einen Kopplungscode; genehmigen Sie ihn mit:
   - `openclaw pairing approve googlechat <code>`
6. Gruppenbereiche erfordern standardmäßig eine @-Erwähnung. Erwähnungen werden anhand von Chat-Annotationen des Typs `USER_MENTION` erkannt, die auf die App verweisen; legen Sie `botUser` fest (z. B. `users/1234567890`), wenn zur Erkennung der Name der Benutzerressource der App benötigt wird.
7. Wenn eine Ausführungs- oder Plugin-Genehmigung über Google Chat gestartet wird und ein stabiler Genehmiger vom Typ `users/<id>` konfiguriert ist, veröffentlicht OpenClaw eine native Genehmigungskarte (`cardsV2`) im ursprünglichen Bereich oder Thread. Die Schaltflächen der Karte enthalten undurchsichtige Callback-Tokens; die manuelle Aufforderung `/approve <id> <decision>` wird nur angezeigt, wenn die native Zustellung nicht verfügbar ist.

### Dauerhafte Verarbeitung eingehender Ereignisse

Nach der Authentifizierung der Anfrage entfernt OpenClaw das Add-on-Autorisierungsobjekt aus dem Speicher und reiht Google-Chat-Ereignisse vom Typ `MESSAGE` dauerhaft in die Warteschlange ein, bevor `200` zurückgegeben wird. Bei einem Persistenzfehler wird `503` zurückgegeben, sodass Google Chat den Vorgang wiederholen kann, anstatt ein möglicherweise verlorenes Ereignis zu bestätigen.

Ausstehende oder wiederholbare Nachrichten überstehen einen Neustart des Gateways, bleiben pro Bereich serialisiert und verwenden den Ressourcennamen der Google-Chat-Nachricht, um doppelte Warteschlangeneinträge zu unterdrücken, solange der aktive oder aufbewahrte Abschlussdatensatz vorhanden ist. Aktionen, die keine Nachrichten sind, verwenden weiterhin ihren bestehenden abgekoppelten Webhook-Pfad und erhalten diese Garantie der dauerhaften Warteschlange nicht. Über die Grenze zwischen Warteschlange und Agent hinweg erfolgt die Zustellung weiterhin mindestens einmal, sodass ein Absturz während der Übergabe einen Durchlauf erneut ausführen kann.

## Ziele

Verwenden Sie diese Bezeichner für die Zustellung und Positivlisten:

- Direktnachrichten: `users/<userId>` (empfohlen).
- Bereiche: `spaces/<spaceId>`.
- Die reine E-Mail-Adresse `name@example.com` ist veränderlich und wird nur für den Abgleich mit der Positivliste verwendet, wenn `channels.googlechat.dangerouslyAllowNameMatching: true`.
- Veraltet: `users/<email>` wird als Benutzer-ID und nicht als E-Mail-Eintrag der Positivliste behandelt.
- Die Präfixe `googlechat:`, `google-chat:` und `gchat:` werden akzeptiert und entfernt.

## Wichtige Konfigurationsoptionen

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // oder serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      appPrincipal: "123456789012345678901", // nur Add-on-Überprüfung; numerische OAuth-Client-ID
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // optional; erleichtert die Erkennung von Erwähnungen
      allowBots: false,
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          enabled: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "Nur kurze Antworten.",
        },
      },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

Hinweise:

- Anmeldedaten des Dienstkontos: `serviceAccountFile` (Pfad), `serviceAccount` (eingebettete JSON-Zeichenfolge oder eingebettetes JSON-Objekt) oder `serviceAccountRef` (Umgebungsvariablen-/Datei-SecretRef). Die Umgebungsvariablen `GOOGLE_CHAT_SERVICE_ACCOUNT` (eingebettetes JSON) und `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (Pfad) gelten nur für das Standardkonto. Konfigurationen mit mehreren Konten verwenden `channels.googlechat.accounts.<id>` mit denselben Schlüsseln, einschließlich des kontospezifischen Schlüssels `serviceAccountRef`.
- Der standardmäßige Webhook-Pfad ist `/googlechat`, wenn `webhookPath` nicht festgelegt ist; alternativ kann `webhookUrl` den Pfad bereitstellen.
- Gruppenschlüssel müssen stabile Bereichs-IDs sein (`spaces/<spaceId>`). Schlüssel mit Anzeigenamen sind veraltet und werden entsprechend protokolliert.
- `dangerouslyAllowNameMatching` aktiviert den Abgleich veränderlicher E-Mail-Hauptkonten für Positivlisten erneut (Kompatibilitätsmodus für Notfälle); der Doctor warnt vor E-Mail-Einträgen.
- Google-Chat-Reaktionsaktionen werden nicht bereitgestellt. Das Plugin verwendet die Dienstkontoauthentifizierung, während Google-Chat-Reaktionsendpunkte eine Benutzerauthentifizierung erfordern. Die bestehende Konfiguration `actions.reactions` wird aus Kompatibilitätsgründen akzeptiert, hat jedoch keine Wirkung.
- Native Genehmigungskarten verwenden Schaltflächenklicks vom Typ Google Chat `cardsV2`, keine Reaktionsereignisse. Genehmiger stammen aus `allowFrom` oder `defaultTo` und müssen stabile numerische Werte vom Typ `users/<id>` sein.
- Nachrichtenaktionen stellen ausschließlich Text vom Typ `send` bereit. Das Hochladen von Anhängen in Google Chat erfordert eine Benutzerauthentifizierung, während dieses Plugin eine Dienstkontoauthentifizierung verwendet. Daher wird das Hochladen ausgehender Dateien nicht bereitgestellt.
- `typingIndicator`: `message` (Standard) veröffentlicht einen Platzhalter vom Typ `_<Bot> is typing..._` und wandelt ihn durch Bearbeitung in die erste Antwort um; `none` deaktiviert ihn; `reaction` erfordert Benutzer-OAuth und fällt derzeit bei der Dienstkontoauthentifizierung unter Protokollierung eines Fehlers auf `message` zurück.
- Eingehende Anhänge (der erste Anhang pro Nachricht) werden über die Chat API in die Medienpipeline heruntergeladen und durch `mediaMaxMb` begrenzt (Standardwert 20).
- Von Bots verfasste Nachrichten werden standardmäßig ignoriert. Mit `allowBots: true` verwenden akzeptierte Bot-Nachrichten den gemeinsamen [Bot-Schleifenschutz](/de/channels/bot-loop-protection): Konfigurieren Sie `channels.defaults.botLoopProtection` und überschreiben Sie ihn anschließend mit `channels.googlechat.botLoopProtection` oder `channels.googlechat.groups.<space>.botLoopProtection`.

Details zur Secrets-Referenz: [Secrets-Verwaltung](/de/gateway/secrets).

## Fehlerbehebung

### 405 Method Not Allowed

Wenn der Google Cloud Logs Explorer Fehler wie diesen anzeigt:

```text
status code: 405, reason phrase: HTTP error response: HTTP/1.1 405 Method Not Allowed
```

Der Webhook-Handler ist nicht registriert. Häufige Ursachen:

1. **Kanal nicht konfiguriert**: Der Abschnitt `channels.googlechat` fehlt. Überprüfen Sie dies mit:

   ```bash
   openclaw config get channels.googlechat
   ```

   Wenn „Config path not found“ zurückgegeben wird, fügen Sie die Konfiguration hinzu (siehe [Wichtige Konfigurationseinstellungen](#config-highlights)).

2. **Plugin nicht aktiviert**: Überprüfen Sie den Plugin-Status:

   ```bash
   openclaw plugins list | grep googlechat
   ```

   Wenn „disabled“ angezeigt wird, fügen Sie `plugins.entries.googlechat.enabled: true` zu Ihrer Konfiguration hinzu.

3. **Gateway nach Konfigurationsänderungen nicht neu gestartet**:

   ```bash
   openclaw gateway restart
   ```

Überprüfen Sie, ob der Kanal ausgeführt wird:

```bash
openclaw channels status
# Sollte Folgendes anzeigen: Google Chat default: enabled, configured, ...
```

### Weitere Probleme

- `openclaw channels status --probe` zeigt Authentifizierungsfehler und eine fehlende Zielgruppenkonfiguration an (`audience` und `audienceType` sind beide erforderlich).
- Wenn keine Nachrichten eingehen, überprüfen Sie die Webhook-URL und die Trigger-Konfiguration der Chat-App.
- Wenn die Erwähnungsbeschränkung Antworten blockiert, setzen Sie `botUser` auf den Namen der Benutzerressource der App und überprüfen Sie `requireMention`.
- `openclaw logs --follow` beim Senden einer Testnachricht zeigt, ob Anfragen das Gateway erreichen.

## Verwandte Themen

- [Kanalübersicht](/de/channels) — alle unterstützten Kanäle
- [Kanalrouting](/de/channels/channel-routing) — Sitzungsrouting für Nachrichten
- [Gateway-Konfiguration](/de/gateway/configuration)
- [Gruppen](/de/channels/groups) — Verhalten von Gruppenchats und Erwähnungsbeschränkung
- [Kopplung](/de/channels/pairing) — DM-Authentifizierung und Kopplungsablauf
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Absicherung
