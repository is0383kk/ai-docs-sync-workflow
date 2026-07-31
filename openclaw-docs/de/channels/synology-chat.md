---
read_when:
    - Synology Chat mit OpenClaw einrichten
    - Debugging des Synology-Chat-Webhook-Routings
summary: Synology-Chat-Webhook-Einrichtung und OpenClaw-Konfiguration
title: Synology Chat
x-i18n:
    generated_at: "2026-07-26T18:15:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat verbindet sich über ein Webhook-Paar mit OpenClaw: Ein ausgehender Synology-Chat-Webhook sendet eingehende Direktnachrichten an das Gateway, und Antworten werden über einen eingehenden Synology-Chat-Webhook zurückgesendet.

Status: offizielles Plugin, separat installiert. Nur Direktnachrichten; Textnachrichten und URL-basierte Dateiübertragungen werden unterstützt.

## Installation

```bash
openclaw plugins install @openclaw/synology-chat
```

Lokaler Checkout (bei Ausführung aus einem Git-Repository):

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

Details: [Plugins](/de/tools/plugin)

## Schnelleinrichtung

1. Installieren Sie das Plugin (siehe oben).
2. In den Synology-Chat-Integrationen:
   - Erstellen Sie einen eingehenden Webhook und kopieren Sie dessen URL.
   - Erstellen Sie einen ausgehenden Webhook mit Ihrem geheimen Token.
3. Richten Sie die URL des ausgehenden Webhooks auf Ihr OpenClaw Gateway:
   - `https://gateway-host/webhook/synology` standardmäßig.
   - Oder Ihren benutzerdefinierten `channels.synology-chat.webhookPath`.
4. Schließen Sie die Einrichtung in OpenClaw ab. Synology Chat wird in beiden Abläufen in derselben Liste zur Kanaleinrichtung angezeigt:
   - Geführt: `openclaw onboard` oder `openclaw channels add`
   - Direkt: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. Starten Sie das Gateway neu und senden Sie dem Synology-Chat-Bot eine Direktnachricht.

Details zur Webhook-Authentifizierung:

- OpenClaw akzeptiert das Token des ausgehenden Webhooks zunächst aus `body.token`, dann aus
  `?token=...` und anschließend aus Headern.
- Akzeptierte Header-Formen:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- Leere oder fehlende Token führen zu einer sicheren Ablehnung.
- Nutzdaten können `application/x-www-form-urlencoded` oder `application/json` sein; `token`, `user_id` und `text` sind erforderlich.

## Dauerhaftigkeit eingehender Nachrichten

Nachdem die Prüfungen des Tokens, der Absenderrichtlinie und des Ratenlimits bestanden wurden, entfernt OpenClaw das Webhook-Token aus dem gespeicherten Umschlag und reiht das Ereignis dauerhaft in die Warteschlange ein, bevor es bestätigt wird. Die Route gibt `204` erst zurück, nachdem dieses Anhängen erfolgreich war; bei einem Persistenzfehler wird `503` zurückgegeben, damit Synology Chat den Vorgang wiederholen kann, anstatt die Nachricht unbemerkt zu verlieren.

Ausstehende oder wiederholbare Ereignisse überstehen einen Neustart des Gateways. Synologys stabile `post_id` unterdrückt doppelte Warteschlangeneinträge, solange der entsprechende aktive oder aufbewahrte Abschlussdatensatz vorhanden ist. Die Zustellung zwischen Warteschlange und Agent erfolgt weiterhin mindestens einmal, sodass ein Absturz an dieser Übergabestelle dennoch eine erneute Ausführung eines Durchlaufs auslösen kann.

Minimale Konfiguration:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## Umgebungsvariablen

Für das Standardkonto können Sie Umgebungsvariablen verwenden:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (durch Kommas getrennt)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

Konfigurationswerte überschreiben Umgebungsvariablen.

`SYNOLOGY_CHAT_INCOMING_URL` und `SYNOLOGY_NAS_HOST` können nicht über eine `.env` des Arbeitsbereichs festgelegt werden; siehe [`.env`-Dateien des Arbeitsbereichs](/de/gateway/security#workspace-env-files).

## Richtlinie und Zugriffssteuerung für Direktnachrichten

- Unterstützte Werte für `dmPolicy`: `allowlist` (Standard), `open` und `disabled`. Synology Chat verfügt über keinen Kopplungsablauf; genehmigen Sie Absender, indem Sie deren numerische Synology-Benutzer-IDs zu `allowedUserIds` hinzufügen.
- `allowedUserIds` akzeptiert eine Liste (oder eine durch Kommas getrennte Zeichenfolge) mit Synology-Benutzer-IDs.
- Im Modus `allowlist` wird eine leere `allowedUserIds`-Liste als Fehlkonfiguration behandelt, und die Webhook-Route wird nicht gestartet.
- `dmPolicy: "open"` erlaubt öffentliche Direktnachrichten nur, wenn `allowedUserIds` den Eintrag `"*"` enthält; bei einschränkenden Einträgen können nur übereinstimmende Benutzer chatten. Auch bei `open` mit einer leeren `allowedUserIds`-Liste wird der Start der Route verweigert.
- `dmPolicy: "disabled"` blockiert Direktnachrichten.
- Die Bindung des Antwortempfängers bleibt standardmäßig an die stabile numerische `user_id` gebunden. `channels.synology-chat.dangerouslyAllowNameMatching: true` ist ein Kompatibilitätsmodus für Notfälle, der die Suche anhand veränderlicher Benutzernamen bzw. Spitznamen für die Antwortzustellung wieder aktiviert.

## Ausgehende Zustellung

Verwenden Sie numerische Synology-Chat-Benutzer-IDs als Ziele. Die Präfixe `synology-chat:`, `synology_chat:` und `synology:` werden akzeptiert.

Beispiele:

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hallo von OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Nochmals hallo"
openclaw message send --channel synology-chat --target synology:123456 --message "Kurzes Präfix"
```

Ausgehender Text wird bei 2000 Zeichen aufgeteilt. Das Senden von Medien wird über URL-basierte Dateizustellung unterstützt: Das NAS lädt die Datei herunter und hängt sie an (max. 32 MB). URLs ausgehender Dateien müssen `http` oder `https` verwenden; private oder anderweitig blockierte Netzwerkziele werden abgelehnt, bevor OpenClaw die URL an den NAS-Webhook weiterleitet.

## Mehrere Konten

Unter `channels.synology-chat.accounts` werden mehrere Synology-Chat-Konten unterstützt.
Jedes Konto kann Token, eingehende URL, Webhook-Pfad, Direktnachrichtenrichtlinie und Limits überschreiben.
Direktnachrichtensitzungen werden nach Konto und Benutzer getrennt, sodass dieselbe numerische `user_id`
in zwei verschiedenen Synology-Konten keinen gemeinsamen Transkriptstatus verwendet.
Weisen Sie jedem aktivierten Konto einen eigenen `webhookPath` zu. OpenClaw lehnt identische doppelte Pfade ab
und verweigert in Konfigurationen mit mehreren Konten den Start benannter Konten, die lediglich einen gemeinsamen Webhook-Pfad erben.
Wenn Sie für ein benanntes Konto absichtlich die alte Vererbung benötigen, legen Sie
`dangerouslyAllowInheritedWebhookPath: true` für dieses Konto oder unter `channels.synology-chat` fest;
identische doppelte Pfade werden jedoch weiterhin sicher abgelehnt. Bevorzugen Sie explizite Pfade pro Konto.

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## Sicherheitshinweise

- Halten Sie `token` geheim und rotieren Sie es, falls es offengelegt wurde.
- Behalten Sie `allowInsecureSsl: false` bei, sofern Sie nicht ausdrücklich einem selbstsignierten Zertifikat eines lokalen NAS vertrauen.
- Eingehende Webhook-Anfragen werden anhand des Tokens verifiziert und pro Absender ratenbegrenzt (`rateLimitPerMinute`, Standard: 30).
- Bei Prüfungen ungültiger Token wird ein zeitkonstanter Vergleich geheimer Werte verwendet und sicher abgelehnt; wiederholte Versuche mit ungültigen Token sperren die Quell-IP vorübergehend.
- Der Text eingehender Nachrichten wird gegen bekannte Prompt-Injection-Muster bereinigt und auf 4000 Zeichen gekürzt.
- Bevorzugen Sie für den Produktivbetrieb `dmPolicy: "allowlist"`.
- Lassen Sie `dangerouslyAllowNameMatching` deaktiviert, sofern Sie nicht ausdrücklich die alte benutzernamenbasierte Antwortzustellung benötigen.
- Lassen Sie `dangerouslyAllowInheritedWebhookPath` deaktiviert, sofern Sie nicht ausdrücklich das Risiko der Weiterleitung über gemeinsam verwendete Pfade in einer Konfiguration mit mehreren Konten akzeptieren.

## Fehlerbehebung

- `Missing required fields (token, user_id, text)`:
  - In den Nutzdaten des ausgehenden Webhooks fehlt eines der erforderlichen Felder
  - Wenn Synology das Token in Headern sendet, stellen Sie sicher, dass das Gateway bzw. der Proxy diese Header beibehält
- `Invalid token`:
  - Das Geheimnis des ausgehenden Webhooks stimmt nicht mit `channels.synology-chat.token` überein
  - Die Anfrage trifft auf das falsche Konto bzw. den falschen Webhook-Pfad
  - Ein Reverse-Proxy hat den Token-Header entfernt, bevor die Anfrage OpenClaw erreichte
- `Rate limit exceeded`:
  - Zu viele Versuche mit ungültigen Token aus derselben Quelle können diese Quelle vorübergehend sperren
  - Für authentifizierte Absender gilt außerdem ein separates Nachrichtenratenlimit pro Benutzer
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`:
  - `dmPolicy="allowlist"` ist aktiviert, es sind jedoch keine Benutzer konfiguriert
- `User not authorized`:
  - Die numerische `user_id` des Absenders ist nicht in `allowedUserIds` enthalten

## Verwandte Themen

- [Kanalübersicht](/de/channels) — alle unterstützten Kanäle
- [Gruppen](/de/channels/groups) — Verhalten von Gruppenchats und Erwähnungssteuerung
- [Kanal-Routing](/de/channels/channel-routing) — Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Härtung
