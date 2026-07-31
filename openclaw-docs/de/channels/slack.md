---
read_when:
    - Slack einrichten oder den Socket-, HTTP- oder Relay-Modus von Slack debuggen
summary: Slack-Einrichtung und Laufzeitverhalten (Socket Mode, HTTP Request URLs und Relay-Modus)
title: Slack
x-i18n:
    generated_at: "2026-07-26T17:39:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0f974ddf8e6965b09cede6a16f171434915a994fa3c1fc744d2350399941bee
    source_path: channels/slack.md
    workflow: 16
---

Slack-Unterstützung umfasst Direktnachrichten und Kanäle über Slack-App-Integrationen. Der Standardtransport ist Socket Mode; HTTP Request URLs werden ebenfalls unterstützt. Der Relay-Modus ist für verwaltete Bereitstellungen vorgesehen, bei denen ein vertrauenswürdiger Router den Slack-Eingang verwaltet.

<CardGroup cols={3}>
  <Card title="Kopplung" icon="link" href="/de/channels/pairing">
    Slack-Direktnachrichten verwenden standardmäßig den Kopplungsmodus.
  </Card>
  <Card title="Slash-Befehle" icon="terminal" href="/de/tools/slash-commands">
    Natives Befehlsverhalten und Befehlskatalog.
  </Card>
  <Card title="Fehlerbehebung für Kanäle" icon="wrench" href="/de/channels/troubleshooting">
    Kanalübergreifende Diagnose- und Reparaturleitfäden.
  </Card>
</CardGroup>

## Transport auswählen

Socket Mode und HTTP Request URLs bieten Funktionsparität für Nachrichten, Slash-Befehle, App Home und Interaktionen. Treffen Sie die Wahl anhand der Bereitstellungsarchitektur, nicht anhand der Funktionen.

| Aspekt                       | Socket Mode (Standard)                                                                                                                               | HTTP Request URLs                                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Öffentliche Gateway-URL      | Nicht erforderlich                                                                                                                                    | Erforderlich (DNS, TLS, Reverse-Proxy oder Tunnel)                                                             |
| Ausgehendes Netzwerk         | Ausgehendes WSS zu `wss-primary.slack.com` muss erreichbar sein                                                                                            | Kein ausgehendes WS; nur eingehendes HTTPS                                                                     |
| Benötigte Tokens             | Bot-Identität: Bot-Token + App-Level Token mit `connections:write`; Benutzeridentität: Benutzer-Token + App-Level Token                               | Bot-Identität: Bot-Token + Signing Secret; Benutzeridentität: Benutzer-Token + Signing Secret                 |
| Entwicklungs-Laptop / hinter Firewall | Funktioniert ohne weitere Konfiguration                                                                                                      | Erfordert einen öffentlichen Tunnel (ngrok, Cloudflare Tunnel, Tailscale Funnel) oder ein Staging-Gateway     |
| Horizontale Skalierung       | Eine Socket-Mode-Sitzung pro App und Host; mehrere Gateways benötigen separate Slack-Apps                                                            | Zustandsloser POST-Handler; mehrere Gateway-Replikate können sich hinter einem Load-Balancer eine App teilen  |
| Mehrere Konten auf einem Gateway | Unterstützt; jedes Konto öffnet ein eigenes WS                                                                                                  | Unterstützt; jedes Konto benötigt einen eindeutigen `webhookPath` (Standard `/slack/events`), damit Registrierungen nicht kollidieren |
| Transport für Slash-Befehle  | Über die WS-Verbindung zugestellt; `slash_commands[].url` wird ignoriert                                                                                  | Slack sendet POST-Anfragen an `slash_commands[].url`; das Feld ist für die Weiterleitung des Befehls erforderlich |
| Signierung von Anfragen      | Nicht verwendet (Authentifizierung erfolgt über das App-Level Token)                                                                                 | Slack signiert jede Anfrage; OpenClaw verifiziert sie mit `signingSecret`                                  |
| Wiederherstellung bei Verbindungsabbruch | Die automatische Wiederverbindung des Slack SDK ist aktiviert; OpenClaw startet fehlgeschlagene Socket-Mode-Sitzungen zusätzlich mit begrenztem Backoff neu. Die Transportoptimierung für Pong-Zeitüberschreitungen gilt. | Es gibt keine persistente Verbindung, die abbrechen kann; Wiederholungsversuche erfolgen pro Anfrage durch Slack |

<Note>
  **Wählen Sie Socket Mode** für Hosts mit einem einzelnen Gateway, Entwicklungs-Laptops und lokale Netzwerke, die `*.slack.com` ausgehend erreichen können, aber kein eingehendes HTTPS akzeptieren können.

**Wählen Sie HTTP Request URLs**, wenn mehrere Gateway-Replikate hinter einem Load-Balancer ausgeführt werden, ausgehendes WSS blockiert ist, aber eingehendes HTTPS erlaubt ist, oder wenn Slack-Webhooks bereits an einem Reverse-Proxy terminieren.
</Note>

<Warning>
  Slack kann für eine App mehrere Socket-Mode-Verbindungen aufrechterhalten und jede Nutzlast an eine beliebige Verbindung zustellen. Separate OpenClaw-Gateways, die sich eine Slack-App teilen, benötigen daher eine gleichwertige Routing- und Autorisierungskonfiguration. Verwenden Sie andernfalls eine separate Slack-App pro Gateway, einen einzelnen Relay-Eingang oder HTTP Request URLs hinter einem Load-Balancer. Siehe [Socket Mode verwenden](https://docs.slack.dev/apis/events-api/using-socket-mode#using-multiple-connections).
</Warning>

### Relay-Modus

Der Relay-Modus trennt den Slack-Eingang vom OpenClaw-Gateway. Ein vertrauenswürdiger Router verwaltet die einzelne Slack-Socket-Mode-Verbindung, wählt ein Ziel-Gateway aus und leitet ein typisiertes Ereignis über einen authentifizierten WebSocket weiter. Das Gateway verwendet weiterhin sein eigenes Bot-Token für ausgehende Aufrufe der Slack Web API.

```json5
{
  channels: {
    slack: {
      mode: "relay",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      relay: {
        url: "wss://router.example.com/gateway/ws",
        authToken: { source: "env", provider: "default", id: "SLACK_RELAY_AUTH_TOKEN" },
        gatewayId: "team-gateway",
      },
    },
  },
}
```

Die Relay-URL muss `wss://` verwenden, sofern sie nicht auf localhost verweist. Behandeln Sie das Bearer-Token und die Routentabelle des Routers als Teil der Slack-Autorisierungsgrenze: Weitergeleitete Ereignisse gelangen als autorisierte Aktivierungen in den normalen Slack-Nachrichtenhandler. Ein vom Router bereitgestellter `slack_identity` im WebSocket-Frame `hello` kann den standardmäßigen ausgehenden Benutzernamen und das Symbol festlegen; eine explizit vom Aufrufer angegebene Identität hat weiterhin Vorrang. Die Relay-Verbindung wird mit demselben begrenzten Backoff-Timing wie Socket Mode erneut hergestellt und löscht die vom Router bereitgestellte Identität bei jeder Trennung.

### Organisationsweite Enterprise-Grid-Installationen

Ein Slack-Konto kann Nachrichten aus jedem Workspace empfangen, der von einer
organisationsweiten Enterprise-Grid-Installation abgedeckt wird. Wählen Sie den
direkten Socket Mode oder HTTP Request URLs; der Relay-Modus wird für
Enterprise-Konten nicht unterstützt. Beide nachstehenden Manifeste mit minimalen
Berechtigungen aktivieren nur den V1-Ereignispfad `message` und `app_mention`,
sofortige Antworten und vom Listener verwaltete Statusreaktionen.

#### Socket Mode

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-Konnektor für OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Lassen Sie die App von einem Enterprise Grid Org Admin oder Org Owner genehmigen,
installieren Sie sie auf Organisationsebene und wählen Sie die Workspaces aus,
die von der Installation abgedeckt werden. Vergewissern Sie sich vor dem Start
von OpenClaw, dass die App in jedem vorgesehenen Workspace verfügbar ist.
Generieren Sie für Socket Mode ein App-Level Token mit `connections:write` und
kopieren Sie anschließend das Bot-Token aus der Organisationsinstallation.
Konfigurieren Sie das Konto, das das organisationsweit installierte Bot-Token
verwendet:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      enterpriseOrgInstall: true,
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

#### HTTP Request URLs

Verwenden Sie den HTTP-Modus, wenn das Gateway über einen öffentlichen
HTTPS-Endpunkt verfügt und keine Socket-Mode-Verbindung öffnet. Ersetzen Sie
die Beispiel-URL durch die öffentliche `webhookPath`-URL des Gateways
(Standard `/slack/events`):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-Konnektor für OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Lassen Sie die App von einem Enterprise Grid Org Admin oder Org Owner genehmigen,
installieren Sie sie auf Organisationsebene und wählen Sie die Workspaces aus,
die von der Installation abgedeckt werden. Nachdem Slack die Request URL
verifiziert hat, kopieren Sie das Bot-Token der Organisationsinstallation und
das **Basic Information -> App Credentials -> Signing Secret** der App.
Konfigurieren Sie das Enterprise-Konto mit demselben Request-URL-Pfad:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      enterpriseOrgInstall: true,
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: {
        source: "env",
        provider: "default",
        id: "SLACK_SIGNING_SECRET",
      },
      webhookPath: "/slack/events",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

Beim Start verifiziert OpenClaw `enterpriseOrgInstall` mit Slack `auth.test`.
Ein organisationsweit installiertes Token ohne das Flag oder ein Workspace-Token
mit dem Flag führt zu einem fehlgeschlagenen Start. Slack bleibt die maßgebliche
Quelle dafür, welche Workspaces die Installation genehmigt haben; OpenClaw
wendet anschließend die konfigurierten Kanal-, Benutzer-, Direktnachrichten-
und Erwähnungsrichtlinien auf jedes zugestellte Ereignis an. Enterprise V1
weist alle von Bots erzeugten Ereignisse `message` und
`app_mention` vor der Weiterleitung zurück, unabhängig von
`allowBots`, da organisationsweite Installationen keine stabile,
Workspace-qualifizierte Bot-Identität zur Vermeidung von Schleifen bereitstellen.

Die Enterprise-Unterstützung ist bewusst auf direkten Socket Mode oder HTTP,
die Ereignisse `message` und `app_mention` sowie deren sofortige
Antworten beschränkt. Relay-Modus, Slash-Befehle, Interaktionen, App Home,
Listener für Reaktionsereignisse, angeheftete Elemente, Slack-Aktionswerkzeuge,
Slack-native Genehmigungen, Bindungen, Zustellung per Warteschlange oder Zeitplan
und proaktives Senden sind für ein Enterprise-Konto nicht verfügbar. Ausgehende
Bestätigungs-, Eingabe- und Statusreaktionen werden über den vom Listener
verwalteten Slack-Client unterstützt und erfordern `reactions:write`;
eingehende Reaktionsbenachrichtigungen und Reaktionsaktionswerkzeuge bleiben
nicht verfügbar.

Sofortige Antworten verwenden das standardmäßige Slack-Zustellungsverhalten für Abschnitte,
Medien, Metadaten, Identitäts-Fallback, Link-Vorschauen und Empfangsbestätigungen wieder, jedoch nur, solange der
validierte, dem Listener zugeordnete Client im aktiven Ereignisdurchlauf verbleibt. Die
In-Memory-Sendewarteschlange und die Datensätze zur Thread-Teilnahme werden nach dem Workspace
dieses Ereignisses partitioniert; der Client selbst wird niemals serialisiert oder persistiert.

Kanalrichtlinienschlüssel und `dm.groupChannels`-Einträge müssen unverarbeitete, stabile Slack-Kanal-IDs oder die
Form `channel:<id>` verwenden. OpenClaw normalisiert beide Formen für den
Laufzeitabgleich zur unverarbeiteten Kanal-ID; die Präfixe `slack:`, `group:` und `mpim:` verhindern den Start.
Benutzerrichtlinieneinträge müssen stabile Slack-Benutzer-IDs verwenden; Namen, Slugs, Anzeigenamen
und E-Mail-Adressen verhindern den Start. IDs müssen das kanonische Slack-Präfix in Großbuchstaben
und den kanonischen Hauptteil verwenden (zum Beispiel `C0123456789` oder `U0123456789`); kleingeschriebene und
verkürzte, ähnlich aussehende Varianten verhindern den Start. Enterprise-Konten können
`dangerouslyAllowNameMatching` nicht aktivieren. Enterprise-Konten können das globale
`mentionPatterns.mode` festlegen, aber `mentionPatterns.allowIn` und
`mentionPatterns.denyIn` verhindern den Start, da einfache Slack-Kanal-IDs nicht
durch einen Workspace qualifiziert sind und in mehreren Workspaces wiederverwendet werden können. Workspace-Installationen
behalten das bestehende bereichsbezogene Verhalten für Erwähnungsmuster bei. Jeder akzeptierte Workspace
erhält eine separate Identität für Routing, Sitzung, Transkript, Deduplizierung, Verlauf und Cache,
selbst wenn sich Slack-IDs überschneiden. Innerhalb des `message`-Streams werden gewöhnliche Benutzernachrichten
und von Benutzern verfasste `file_share`-Ereignisse unterstützt; andere Nachrichtenuntertypen werden
vor der Autorisierung oder der Verarbeitung von Systemereignissen abgelehnt.

Enterprise-Direktnachrichten müssen entweder deaktiviert sein (`dm.enabled=false` oder
`dmPolicy="disabled"`) oder mit `dmPolicy="open"` ausdrücklich geöffnet werden und
eine wirksame Konto-`allowFrom` enthalten, die das Literal `"*"` umfasst. Eine leere
Zulassungsliste oder benutzerspezifische IDs ohne `"*"` verhindern den Start. Kopplung und
benutzerspezifische Zulassungslisten für Direktnachrichten werden abgelehnt, da Slack-Benutzer-IDs in diesen
Autorisierungsspeichern nicht durch einen Workspace qualifiziert sind. Die Richtlinien für Kanäle und Absender
gelten weiterhin für Kanalnachrichten.

## Installation

```bash
openclaw plugins install @openclaw/slack
```

`plugins install` registriert und aktiviert das Plugin. Es bewirkt nichts, bis Sie die Slack-App und die nachfolgenden Kanaleinstellungen konfigurieren. Allgemeine Regeln zur Plugin-Installation finden Sie unter [Plugins](/de/tools/plugin).

## Schnelleinrichtung

Die Manifeste in diesem Abschnitt erstellen eine Workspace-bezogene Installation. Verwenden Sie für eine
organisationsweite Installation in einer Enterprise-Grid-Organisation stattdessen das dedizierte
[organisationsweite Manifest und den entsprechenden Ablauf](#enterprise-grid-org-wide-installs).

<Tabs>
  <Tab title="Socket-Modus (Standard)">
    <Steps>
      <Step title="Neue Slack-App erstellen">
        Öffnen Sie [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → wählen Sie Ihren Workspace aus → fügen Sie eines der nachfolgenden Manifeste ein → **Next** → **Create**.

        <CodeGroup>

```json Empfohlen
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-Konnektor für OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw verbindet Unterhaltungen in der Slack-Agentenansicht mit OpenClaw-Agenten.",
      "suggested_prompts": [
        { "title": "Was können Sie tun?", "message": "Wobei können Sie mir helfen?" },
        {
          "title": "Diesen Kanal zusammenfassen",
          "message": "Fassen Sie die jüngsten Aktivitäten in diesem Kanal zusammen."
        },
        { "title": "Antwort entwerfen", "message": "Helfen Sie mir, eine Antwort zu entwerfen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Eine Nachricht an OpenClaw senden",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-Konnektor für OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw verbindet Unterhaltungen in der Slack-Agentenansicht mit OpenClaw-Agenten.",
      "suggested_prompts": [
        { "title": "Was können Sie tun?", "message": "Wobei können Sie mir helfen?" },
        {
          "title": "Diesen Kanal zusammenfassen",
          "message": "Fassen Sie die jüngsten Aktivitäten in diesem Kanal zusammen."
        },
        { "title": "Antwort entwerfen", "message": "Helfen Sie mir, eine Antwort zu entwerfen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Eine Nachricht an OpenClaw senden",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        </CodeGroup>

        <Note>
          **Empfohlen** entspricht dem vollständigen Funktionsumfang des Slack-Plugins: App Home, Slash-Befehle, Dateien, Reaktionen, Pins, Gruppen-Direktnachrichten sowie Lesezugriffe auf Emojis und Benutzergruppen. Wählen Sie **Minimal**, wenn die Workspace-Richtlinie Scopes einschränkt — diese Variante deckt Direktnachrichten, Kanal-/Gruppenverläufe, Erwähnungen und Slash-Befehle ab, lässt jedoch Dateien, Reaktionen, Pins, Gruppen-Direktnachrichten (`mpim:*`), `emoji:read` und `usergroups:read` weg. Unter [Checkliste für Manifest und Scopes](#manifest-and-scope-checklist) finden Sie die Begründung für jeden Scope und additive Optionen wie zusätzliche Slash-Befehle.
        </Note>

        Nachdem Slack die App erstellt hat:

        - **Basic Information -> App-Level Tokens -> Generate Token and Scopes**: Fügen Sie `connections:write` hinzu, speichern Sie und kopieren Sie das Token auf App-Ebene.
        - **Install App -> Install to Workspace**: Kopieren Sie das OAuth-Token des Bot-Benutzers.

      </Step>

      <Step title="OpenClaw konfigurieren">

        Empfohlene SecretRef-Einrichtung:

```bash
export SLACK_APP_TOKEN=slack-app-token-example
export SLACK_BOT_TOKEN=slack-bot-token-example
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        Umgebungsvariablen-Fallback (nur Standardkonto):

```bash
SLACK_APP_TOKEN=slack-app-token-example
SLACK_BOT_TOKEN=slack-bot-token-example
```

      </Step>

      <Step title="Gateway starten">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP-Anfrage-URLs">
    <Steps>
      <Step title="Neue Slack-App erstellen">
        Öffnen Sie [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → wählen Sie Ihren Workspace aus → fügen Sie eines der nachfolgenden Manifeste ein → ersetzen Sie `https://gateway-host.example.com/slack/events` durch Ihre öffentliche Gateway-URL → **Next** → **Create**.

        <CodeGroup>

```json Empfohlen
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-Konnektor für OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw verbindet Unterhaltungen in der Slack-Agentenansicht mit OpenClaw-Agenten.",
      "suggested_prompts": [
        { "title": "Was können Sie tun?", "message": "Wobei können Sie mir helfen?" },
        {
          "title": "Diesen Kanal zusammenfassen",
          "message": "Fassen Sie die jüngsten Aktivitäten in diesem Kanal zusammen."
        },
        { "title": "Antwort entwerfen", "message": "Helfen Sie mir, eine Antwort zu entwerfen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Eine Nachricht an OpenClaw senden",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-Connector für OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw verbindet Unterhaltungen in der Slack Agent View mit OpenClaw-Agenten.",
      "suggested_prompts": [
        { "title": "Was können Sie tun?", "message": "Wobei können Sie mir helfen?" },
        {
          "title": "Diesen Channel zusammenfassen",
          "message": "Fassen Sie die letzten Aktivitäten in diesem Channel zusammen."
        },
        { "title": "Antwort entwerfen", "message": "Helfen Sie mir, eine Antwort zu entwerfen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Eine Nachricht an OpenClaw senden",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

        </CodeGroup>

        <Note>
          **Empfohlen** entspricht dem vollständigen Funktionsumfang des Slack-Plugins; **Minimal** verzichtet für restriktive Workspaces auf Dateien, Reaktionen, Pins, Gruppen-DMs (`mpim:*`), `emoji:read` und `usergroups:read`. Unter [Checkliste für Manifest und Berechtigungsbereiche](#manifest-and-scope-checklist) finden Sie die Begründung für jeden Berechtigungsbereich.
        </Note>

        <Info>
          Die drei URL-Felder (`slash_commands[].url`, `event_subscriptions.request_url` und `interactivity.request_url` / `message_menu_options_url`) verweisen alle auf denselben OpenClaw-Endpunkt. Das Manifest-Schema von Slack verlangt, dass sie separat benannt werden, OpenClaw leitet jedoch nach Nutzlasttyp weiter, sodass ein einzelner `webhookPath` (Standard: `/slack/events`) ausreicht. Slash-Befehle ohne `slash_commands[].url` bleiben im HTTP-Modus ohne Fehlermeldung wirkungslos.
        </Info>

        Nachdem Slack die App erstellt hat:

        - **Basic Information → App Credentials**: Kopieren Sie das **Signing Secret** zur Verifizierung von Anfragen.
        - **Install App -> Install to Workspace**: Kopieren Sie das Bot User OAuth Token.

      </Step>

      <Step title="OpenClaw konfigurieren">

        Empfohlene SecretRef-Einrichtung:

```bash
export SLACK_BOT_TOKEN=slack-bot-token-example
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        <Note>
        Verwenden Sie für HTTP mit mehreren Konten eindeutige Webhook-Pfade

        Weisen Sie jedem Konto einen eigenen `webhookPath` (Standard: `/slack/events`) zu, damit sich die Registrierungen nicht überschneiden.
        </Note>

      </Step>

      <Step title="Gateway starten">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Benutzeridentität (als reale Person posten)

Mit der Benutzeridentität kann OpenClaw als die Person lesen und posten, die die Slack-App autorisiert. `userToken` ist die handelnde Identität; eine zugehörige Slack-App überträgt den Datenverkehr der Events API über Socket Mode oder eine HTTP Request URL. Die zugehörige App benötigt weder einen Bot-Benutzer noch ein Bot-Token.

Richten Sie die zugehörige App wie folgt ein:

1. Fügen Sie unter **OAuth & Permissions -> User Token Scopes** diese benutzerbezogenen Berechtigungen hinzu:

   - Verlauf: `channels:history`, `groups:history`, `im:history`, `mpim:history`
   - Unterhaltungssuche: `channels:read`, `groups:read`, `im:read`, `mpim:read`
   - Personen: `users:read`
   - Posten: `chat:write` (Nachrichten werden als der autorisierende Benutzer gepostet)
   - DMs öffnen: `im:write`, `mpim:write`

2. Fügen Sie unter **Event Subscriptions -> Subscribe to events on behalf of users** diese Benutzerereignisse hinzu. Fügen Sie sie nicht ausschließlich der Liste der Bot-Ereignisse hinzu:

   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`

3. Wählen Sie eine Ereignisübertragung:

   - **Socket Mode:** Aktivieren Sie Socket Mode und erstellen Sie ein Token auf App-Ebene mit `connections:write`. Konfigurieren Sie es als `appToken`.
   - **HTTP Request URL:** Richten Sie Event Subscriptions auf den öffentlichen Slack-Endpunkt von OpenClaw und kopieren Sie **Basic Information -> App Credentials -> Signing Secret**. Konfigurieren Sie es als `signingSecret`.

4. Installieren oder installieren Sie die App erneut, autorisieren Sie sie als die vorgesehene Person und kopieren Sie das resultierende Benutzer-OAuth-Token nach `userToken`.

Socket-Mode-Konfiguration:

```json5
{
  channels: {
    slack: {
      identity: "user",
      userToken: "<xoxp>",
      appToken: "<xapp>",
    },
  },
}
```

Konfiguration der HTTP Request URL:

```json5
{
  channels: {
    slack: {
      identity: "user",
      mode: "http",
      userToken: "<xoxp>",
      signingSecret: "<signing-secret>",
      webhookPath: "/slack/events",
    },
  },
}
```

<Warning>
  DMs und Gruppen-DMs funktionieren nur über das oben beschriebene benutzerbezogene Ereignisabonnement. Ein Bot kann weder einer menschlichen 1:1-DM beitreten noch in eine bestehende Gruppen-DM eingefügt werden. Die zugehörige App fungiert als unsichtbare Infrastruktur: Andere Slack-Mitglieder sehen Nachrichten von der autorisierenden Person, nicht von einem OpenClaw-Bot.
</Warning>

OpenClaw verwirft automatisch benutzerbezogene Nachrichtenereignisse, die von der ermittelten menschlichen Identität stammen, sodass gesendete Nachrichten keine Antworten an sich selbst auslösen.

## Transportoptimierung für Socket Mode

OpenClaw setzt das Pong-Timeout des Slack-SDK-Clients für Socket Mode standardmäßig auf 15 Sekunden. Überschreiben Sie die Transporteinstellungen nur, wenn eine Workspace- oder hostspezifische Optimierung erforderlich ist:

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

Verwenden Sie dies nur für Socket-Mode-Workspaces, die Zeitüberschreitungen bei Slack-WebSocket-Pongs oder Server-Pings protokollieren, oder auf Hosts mit bekannter Überlastung der Ereignisschleife ausgeführt werden. `clientPingTimeout` ist die Wartezeit auf das Pong, nachdem das SDK einen Client-Ping gesendet hat; `serverPingTimeout` ist die Wartezeit auf Server-Pings von Slack. App-Nachrichten und Ereignisse bleiben Anwendungszustand und sind keine Signale für die Funktionsfähigkeit des Transports.

Hinweise:

- `socketMode` wird im HTTP-Request-URL-Modus ignoriert.
- Die grundlegenden `channels.slack.socketMode`-Einstellungen gelten für alle Slack-Konten, sofern sie nicht überschrieben werden. Kontospezifische Überschreibungen verwenden `channels.slack.accounts.<accountId>.socketMode`; da es sich um eine Objektüberschreibung handelt, müssen Sie alle Socket-Optimierungsfelder angeben, die für dieses Konto gelten sollen.
- Nur `clientPingTimeout` besitzt einen OpenClaw-Standardwert (`15000`). `serverPingTimeout` und `pingPongLoggingEnabled` werden nur dann an das Slack-SDK übergeben, wenn sie konfiguriert sind.
- Die Wiederholungsverzögerung beim Neustart von Socket Mode beginnt bei etwa 2 Sekunden und ist auf etwa 30 Sekunden begrenzt. Behebbare Fehler beim Start, beim Warten auf den Start und bei Verbindungsabbrüchen werden wiederholt, bis der Channel beendet wird. Dauerhafte Konto- und Anmeldedatenfehler wie ungültige Authentifizierung, widerrufene Tokens oder fehlende Berechtigungsbereiche schlagen schnell fehl, statt unbegrenzt erneut versucht zu werden.

## Checkliste für Manifest und Berechtigungsbereiche

Das grundlegende Slack-App-Manifest ist für Socket Mode und HTTP Request URLs identisch. Nur der `settings`-Block (und der `url` des Slash-Befehls) unterscheidet sich.

Grundlegendes Manifest (Standard für Socket Mode):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-Connector für OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw verbindet Unterhaltungen in der Slack Agent View mit OpenClaw-Agenten.",
      "suggested_prompts": [
        { "title": "Was können Sie tun?", "message": "Wobei können Sie mir helfen?" },
        {
          "title": "Diesen Channel zusammenfassen",
          "message": "Fassen Sie die letzten Aktivitäten in diesem Channel zusammen."
        },
        { "title": "Antwort entwerfen", "message": "Helfen Sie mir, eine Antwort zu entwerfen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Eine Nachricht an OpenClaw senden",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

Ersetzen Sie für den **HTTP-Request-URL-Modus** `settings` durch die HTTP-Variante und fügen Sie jedem Slash-Befehl `url` hinzu. Eine öffentliche URL ist erforderlich:

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Eine Nachricht an OpenClaw senden",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### Zusätzliche Manifest-Einstellungen

Stellen Sie verschiedene Funktionen bereit, die die obigen Standardeinstellungen erweitern.

Das Standardmanifest aktiviert den Tab **Home** in der Slack App Home und abonniert `app_home_opened`. Wenn ein Workspace-Mitglied den Tab Home öffnet, veröffentlicht OpenClaw eine sichere Standardansicht für Home mit `views.publish`; sie enthält weder Konversationsdaten noch private Konfiguration. Wenn der Modus für einen einzelnen Slash-Befehl aktiviert ist, verwendet der Befehlshinweis `channels.slack.slashCommand.name`; Installationen mit nativen Befehlen oder ohne Slash-Befehle lassen diesen Hinweis weg. Der Tab **Messages** bleibt für Slack-DMs aktiviert. Neue Apps verwenden Slack Agent View über `features.agent_view`, `assistant:write` und `app_context_changed`. Jeder sichtbare Agent-View-Stamm wird an eine eigene OpenClaw-Thread-Sitzung weitergeleitet, und die geordneten aktiven Ansichts-Entitäten von Slack erreichen den Agenten ausschließlich als nicht vertrauenswürdiger Kontext.

Bestehende Apps, die bereits `features.assistant_view` verwenden, können ihr aktuelles Manifest beibehalten. OpenClaw verarbeitet für diese Installationen weiterhin `assistant_thread_started` und `assistant_thread_context_changed`. Slack macht die Migration von Assistant View zu Agent View unumkehrbar und verlangt anschließend eine vollständige Aktualisierung durch die Benutzer. Ersetzen Sie daher `assistant_view` in einer bestehenden App erst, wenn Sie den gesamten Workspace migrieren möchten.

<AccordionGroup>
  <Accordion title="Optionale native Slash-Befehle">

    Mehrere [native Slash-Befehle](#commands-and-slash-behavior) können mit folgenden Besonderheiten anstelle eines einzelnen konfigurierten Befehls verwendet werden:

    - Verwenden Sie `/agentstatus` anstelle von `/status`, da der Befehl `/status` reserviert ist.
    - In einer Slack-App können gleichzeitig höchstens 25 Slash-Befehle registriert werden (Limit der Slack-Plattform).

    OpenClaw registriert Handler für aktivierte native Befehle, die Einträge im Slack-Manifest werden jedoch weiterhin von Administratoren verwaltet und nicht zur Laufzeit synchronisiert. Fügen Sie `/login` manuell zum Manifest hinzu; das folgende Beispiel enthält diesen Befehl anstelle des optionalen Alias `/side`, damit die Anzahl bei 25 Befehlen bleibt. `/login` kann überall angezeigt werden, gibt Kopplungscodes jedoch nur in privaten Chats oder in der Web-UI aus.

    Ersetzen Sie Ihren vorhandenen Abschnitt `features.slash_commands` durch eine Teilmenge der [verfügbaren Befehle](/de/tools/slash-commands#command-list):

    <Tabs>
      <Tab title="Socket Mode (Standard)">

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Neue Sitzung starten",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "Aktuelle Sitzung zurücksetzen"
    },
    {
      "command": "/compact",
      "description": "Sitzungskontext komprimieren",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "Aktuellen Lauf stoppen"
    },
    {
      "command": "/session",
      "description": "Ablauf der Thread-Bindung verwalten",
      "usage_hint": "inaktiv <duration|off> oder maximales Alter <duration|off>"
    },
    {
      "command": "/think",
      "description": "Denkstufe festlegen",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "Ausführliche Ausgabe umschalten",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "Schnellmodus anzeigen oder festlegen",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "Sichtbarkeit der Schlussfolgerungen umschalten",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "Erweiterten Modus umschalten",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "Exec-Standardwerte anzeigen oder festlegen",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/approve",
      "description": "Ausstehende Genehmigungsanfragen genehmigen oder ablehnen",
      "usage_hint": "<id> <decision>"
    },
    {
      "command": "/model",
      "description": "Modell anzeigen oder festlegen",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "Provider/Modelle auflisten",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "Kurze Hilfeübersicht anzeigen"
    },
    {
      "command": "/commands",
      "description": "Generierten Befehlskatalog anzeigen"
    },
    {
      "command": "/tools",
      "description": "Anzeigen, was der aktuelle Agent derzeit verwenden kann",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "Laufzeitstatus einschließlich Provider-Nutzung/Kontingent anzeigen, sofern verfügbar"
    },
    {
      "command": "/tasks",
      "description": "Aktive/kürzlich ausgeführte Hintergrundaufgaben der aktuellen Sitzung auflisten"
    },
    {
      "command": "/context",
      "description": "Erläutern, wie der Kontext zusammengestellt wird",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "Ihre Absenderidentität anzeigen"
    },
    {
      "command": "/skill",
      "description": "Skill anhand des Namens ausführen",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "Nebenfrage stellen, ohne den Sitzungskontext zu ändern",
      "usage_hint": "<question>"
    },
    {
      "command": "/login",
      "description": "Codex-Anmeldung koppeln",
      "usage_hint": "[codex|openai]"
    },
    {
      "command": "/usage",
      "description": "Nutzungsfußzeile steuern oder Kostenzusammenfassung anzeigen",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      </Tab>
      <Tab title="HTTP-Anfrage-URLs">
        Verwenden Sie dieselbe Liste `slash_commands` wie oben für Socket Mode und fügen Sie jedem Eintrag `"url": "https://gateway-host.example.com/slack/events"` hinzu. Beispiel:

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Neue Sitzung starten",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "Kurze Hilfeübersicht anzeigen",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        Wiederholen Sie diesen Wert `url` bei jedem Befehl in der Liste.

      </Tab>
    </Tabs>

  </Accordion>
  <Accordion title="Optionale Urheberschafts-Berechtigungsbereiche (Schreibvorgänge)">
    Fügen Sie den Bot-Berechtigungsbereich `chat:write.customize` hinzu, wenn ausgehende Nachrichten die Identität des aktiven Agenten (benutzerdefinierter Benutzername und benutzerdefiniertes Symbol) anstelle der Standardidentität der Slack-App verwenden sollen.

    Wenn Sie ein Emoji-Symbol verwenden, erwartet Slack die Syntax `:emoji_name:`.

  </Accordion>
  <Accordion title="Optionale Benutzer-Token-Berechtigungsbereiche (Lesevorgänge)">
    Wenn Sie `channels.slack.userToken` konfigurieren, sind folgende Leseberechtigungsbereiche üblich:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (wenn Sie auf Lesezugriffe über die Slack-Suche angewiesen sind)

  </Accordion>
</AccordionGroup>

## Token-Modell

- Die Bot-Identität (Standard) erfordert `botToken` + `appToken` für Socket Mode oder `botToken` + `signingSecret` für den HTTP-Modus.
- Die Benutzeridentität erfordert `userToken` + `appToken` für Socket Mode oder `userToken` + `signingSecret` für den HTTP-Modus. Sie verwendet kein Bot-Token.
- Der Relay-Modus erfordert `botToken` sowie `relay.url`, `relay.authToken` und `relay.gatewayId`; er verwendet weder ein App-Token noch ein Signaturgeheimnis.
- `botToken`, `appToken`, `signingSecret`, `relay.authToken` und `userToken` akzeptieren Klartext-
  Zeichenfolgen oder SecretRef-Objekte.
- Konfigurations-Token überschreiben den Fallback auf Umgebungsvariablen.
- Der Fallback über die Umgebungsvariablen `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` und `SLACK_USER_TOKEN` gilt jeweils nur für das Standardkonto.
- `userToken` verwendet standardmäßig schreibgeschütztes Verhalten (`userTokenReadOnly: true`).

Verhalten der Statusmomentaufnahme:

- Die Slack-Kontoprüfung verfolgt für jede Anmeldeinformation die Felder `*Source` und `*Status`
  (`botToken`, `appToken`, `signingSecret`, `userToken`).
- Der Status lautet `available`, `configured_unavailable` oder `missing`.
- `configured_unavailable` bedeutet, dass das Konto über SecretRef
  oder eine andere nicht eingebettete Quelle für Geheimnisse konfiguriert ist, der aktuelle Befehls-/Laufzeitpfad
  den tatsächlichen Wert jedoch nicht auflösen konnte.
- Im HTTP-Modus ist `signingSecretStatus` enthalten. Socket Mode verwendet
  `botTokenStatus` + `appTokenStatus` für die Bot-Identität und
  `userTokenStatus` + `appTokenStatus` für die Benutzeridentität.

<Tip>
Für die Bot-Identität können Aktionen und Verzeichnis-Lesevorgänge vorzugsweise ein optionales Benutzer-Token verwenden; Schreibvorgänge verwenden weiterhin das Bot-Token, sofern `userTokenReadOnly: false` keinen Fallback erlaubt. Für `identity: "user"` verwenden Lese- und Schreibvorgänge immer `userToken`.
</Tip>

## Aktionen und Sperren

Slack-Aktionen werden durch `channels.slack.actions.*` gesteuert.

Verfügbare Aktionsgruppen in den aktuellen Slack-Werkzeugen:

| Gruppe     | Standard   |
| ---------- | ---------- |
| messages   | aktiviert  |
| reactions  | aktiviert  |
| pins       | aktiviert  |
| memberInfo | aktiviert  |
| emojiList  | aktiviert  |

Die aktuellen Slack-Nachrichtenaktionen umfassen `send`, `upload-file`, `download-file`, `read`, `edit`, `delete`, `pin`, `unpin`, `list-pins`, `member-info` und `emoji-list`. `download-file` akzeptiert Slack-Datei-IDs, die in Platzhaltern für eingehende Dateien angezeigt werden, und gibt für Bilder eine Bildvorschau oder für andere Dateitypen lokale Dateimetadaten zurück.

## Zugriffssteuerung und Routing

<Tabs>
  <Tab title="DM-Richtlinie">
    `channels.slack.dmPolicy` steuert den DM-Zugriff. `channels.slack.allowFrom` ist die kanonische DM-Zulassungsliste.

    - `pairing` (Standard)
    - `allowlist`
    - `open` (erfordert, dass `channels.slack.allowFrom` den Wert `"*"` enthält)
    - `disabled`

    DM-Flags:

    - `dm.enabled` (standardmäßig true)
    - `channels.slack.allowFrom`
    - `dm.allowFrom` (veraltet)
    - `dm.groupEnabled` (Gruppen-DMs standardmäßig false)
    - `dm.groupChannels` (optionale MPIM-Zulassungsliste)

    Rangfolge bei mehreren Konten:

    - `channels.slack.accounts.default.allowFrom` gilt nur für das Konto `default`.
    - Benannte Konten übernehmen `channels.slack.allowFrom`, wenn ihr eigenes `allowFrom` nicht festgelegt ist.
    - Benannte Konten übernehmen `channels.slack.accounts.default.allowFrom` nicht.

    Die veralteten Werte `channels.slack.dm.policy` und `channels.slack.dm.allowFrom` werden aus Kompatibilitätsgründen weiterhin gelesen. `openclaw doctor --fix` migriert sie zu `dmPolicy` und `allowFrom`, sofern dies ohne Änderung des Zugriffs möglich ist.

    Die Kopplung in DMs verwendet `openclaw pairing approve slack <code>`.

  </Tab>

  <Tab title="Kanalrichtlinie">
    `channels.slack.groupPolicy` steuert die Kanalverarbeitung:

    - `open`
    - `allowlist`
    - `disabled`

    Die Kanal-Zulassungsliste befindet sich unter `channels.slack.channels` und **muss stabile Slack-Kanal-IDs** (beispielsweise `C12345678`) als Konfigurationsschlüssel verwenden.

    Laufzeithinweis: Wenn `channels.slack` vollständig fehlt (Einrichtung ausschließlich über Umgebungsvariablen), greift die Laufzeit auf `groupPolicy="allowlist"` zurück und protokolliert eine Warnung (selbst wenn `channels.defaults.groupPolicy` festgelegt ist).

    Namens-/ID-Auflösung:

    - Einträge der Kanal-Zulassungsliste und der DM-Zulassungsliste werden beim Start aufgelöst, sofern der Token-Zugriff dies zulässt
    - Nicht aufgelöste Einträge mit Kanalnamen werden wie konfiguriert beibehalten, aber standardmäßig beim Routing ignoriert
    - Die Autorisierung eingehender Nachrichten und das Kanal-Routing erfolgen standardmäßig primär anhand der ID; der direkte Abgleich von Benutzernamen/Slugs erfordert `channels.slack.dangerouslyAllowNameMatching: true`

    <Warning>
    Namensbasierte Schlüssel (`#channel-name` oder `channel-name`) stimmen unter `groupPolicy: "allowlist"` **nicht** überein. Die Kanalsuche erfolgt standardmäßig zuerst anhand der ID. Daher wird ein namensbasierter Schlüssel niemals erfolgreich weitergeleitet, und alle Nachrichten in diesem Kanal werden ohne Hinweis blockiert. Dies unterscheidet sich von `groupPolicy: "open"`, wo der Kanalschlüssel für die Weiterleitung nicht erforderlich ist und ein namensbasierter Schlüssel zu funktionieren scheint.

    Verwenden Sie als Schlüssel immer die Slack-Kanal-ID. So finden Sie sie: Klicken Sie in Slack mit der rechten Maustaste auf den Kanal → **Copy link** — die ID (`C...`) steht am Ende der URL.

    Richtig:

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```

    Falsch (unter `groupPolicy: "allowlist"` ohne Hinweis blockiert):

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```
    </Warning>

  </Tab>

  <Tab title="Erwähnungen und Kanalbenutzer">
    Kanalnachrichten erfordern standardmäßig eine Erwähnung.

    Quellen für Erwähnungen:

    - explizite App-Erwähnung (`<@botId>`)
    - Slack-Benutzergruppenerwähnung (`<!subteam^S...>`), wenn der Bot-Benutzer Mitglied dieser Benutzergruppe ist; erfordert `usergroups:read`
    - Regex-Muster für Erwähnungen (`agents.entries.*.groupChat.mentionPatterns`, ersatzweise `messages.groupChat.mentionPatterns`)
    - Antworten auf die eigene Slack-Nachricht des Bots (`implicitMentions.replyToBot`)
    - Folgenachrichten in Threads, an denen der Bot beteiligt war (`implicitMentions.threadParticipation`)

    Steuerelemente pro Kanal (`channels.slack.channels.<id>`; Namen nur über die Auflösung beim Start oder `dangerouslyAllowNameMatching`):

    - `requireMention`
    - `ignoreOtherMentions`
    - `replyToMode` (`off|first|all|batched`; überschreibt den Antwortmodus des Kontos bzw. Chat-Typs für diesen Kanal)
    - `users` (Zulassungsliste)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - `toolsBySender`-Schlüsselformat: `channel:`, `id:`, `e164:`, `username:`, `name:` oder der Platzhalter `"*"`
      (alte Schlüssel ohne Präfix werden weiterhin ausschließlich `id:` zugeordnet)

    `ignoreOtherMentions` (Standardwert `false`) verwirft Kanalnachrichten, die einen anderen Benutzer oder eine andere Benutzergruppe erwähnen, aber nicht diesen Bot. DMs und Gruppen-DMs (MPIMs) sind davon nicht betroffen. Der Filter erfordert eine über `auth.test` aufgelöste Bot-Benutzer-ID. Wenn diese Identität nicht verfügbar ist (beispielsweise bei einer Identität, die ausschließlich ein Benutzer-Token verwendet), bleibt die Sperre offen und Nachrichten werden unverändert weitergeleitet.

    `allowBots` ist bei Kanälen und privaten Kanälen restriktiv: Von Bots verfasste Raumnachrichten werden nur akzeptiert, wenn der sendende Bot ausdrücklich in der `users`-Zulassungsliste dieses Raums aufgeführt ist oder wenn derzeit mindestens eine explizite Slack-Eigentümer-ID aus `channels.slack.allowFrom` Mitglied des Raums ist. Platzhalter und Eigentümereinträge mit Anzeigenamen erfüllen die Anforderung der Eigentümeranwesenheit nicht. Zur Prüfung der Eigentümeranwesenheit wird Slack `conversations.members` verwendet. Stellen Sie sicher, dass die App über den entsprechenden Leseberechtigungsumfang für den Raumtyp verfügt (`channels:read` für öffentliche Kanäle, `groups:read` für private Kanäle). Wenn die Mitgliederabfrage fehlschlägt, verwirft OpenClaw die vom Bot verfasste Raumnachricht.

    Akzeptierte, von Bots verfasste Slack-Nachrichten verwenden den gemeinsamen [Schutz vor Bot-Schleifen](/de/channels/bot-loop-protection). Konfigurieren Sie `channels.defaults.botLoopProtection` für das Standardbudget und überschreiben Sie es anschließend mit `channels.slack.botLoopProtection` oder `channels.slack.channels.<id>.botLoopProtection`, wenn ein Workspace oder Kanal ein anderes Limit benötigt.

  </Tab>
</Tabs>

## Threads, Sitzungen und Antwort-Tags

- DMs werden als `direct`, Kanäle als `channel` und MPIMs als `group` weitergeleitet.
- Slack-Routenbindungen akzeptieren unverarbeitete Peer-IDs sowie Slack-Zielformen wie `channel:C12345678`, `user:U12345678` und `<@U12345678>`.
- Mit dem Standardwert `session.dmScope=main` werden gewöhnliche Slack-DMs in der Hauptsitzung des Agenten zusammengeführt. Wurzeln der Agent View und vorhandene Threads der Assistant View bleiben als `:thread:<threadTs>`-Sitzungen isoliert.
- Kanalsitzungen: `agent:<agentId>:slack:channel:<channelId>`.
- Gewöhnliche Kanalnachrichten auf oberster Ebene verbleiben in der kanalspezifischen Sitzung, auch wenn `replyToMode` nicht `off` ist.
- Antworten in Threads von Slack-Kanälen, MPIMs, Agent View und Assistant View verwenden die übergeordnete Slack-`thread_ts` für Sitzungssuffixe (`:thread:<threadTs>`). Gewöhnliche DM-Antwort-Threads bleiben eine UI-Funktion der zugrunde liegenden DM-Sitzung.
- OpenClaw übernimmt eine geeignete Kanalwurzel auf oberster Ebene in `agent:<agentId>:slack:channel:<channelId>:thread:<rootTs>`, wenn von dieser Wurzel erwartet wird, dass sie einen sichtbaren Slack-Thread beginnt, sodass die Wurzel und spätere Thread-Antworten dieselbe OpenClaw-Sitzung verwenden. Dies gilt für `app_mention`-Ereignisse, Übereinstimmungen mit expliziten Bot-Erwähnungen oder konfigurierten Erwähnungsmustern sowie für `requireMention: false`-Kanäle mit einem `replyToMode`, das nicht `off` ist.
- Der Standardwert von `channels.slack.thread.historyScope` ist `thread`; der Standardwert von `thread.inheritParent` ist `false`.
- `channels.slack.thread.initialHistoryLimit` steuert, wie viele vorhandene Thread-Nachrichten beim Start einer neuen Thread-Sitzung abgerufen werden (Standardwert `20`; zum Deaktivieren auf `0` setzen).
- `channels.slack.implicitMentions.replyToBot` steuert, ob eine Antwort auf die eigene Nachricht des Bots die Erwähnungsanforderung umgeht (Standardwert `true`).
- `channels.slack.implicitMentions.threadParticipation` steuert, ob Folgenachrichten in einem Thread, in dem der Bot geantwortet hat, die Erwähnungsanforderung umgehen (Standardwert `true`). Setzen Sie den Wert auf `false`, um in diesen Folgenachrichten eine neue explizite Erwähnung zu verlangen. `openclaw doctor --fix` migriert den früheren Schlüssel `channels.slack.thread.requireExplicitMention` zu diesem positiven kanonischen Flag.
- Kontospezifische Überschreibungen befinden sich unter `channels.slack.accounts.<id>.implicitMentions`; gemeinsame Standardwerte unter `channels.defaults.implicitMentions`.

Steuerelemente für Antwort-Threads:

- `channels.slack.channels.<id>.replyToMode`: kanalspezifische Überschreibung für Nachrichten in Slack-Kanälen und privaten Kanälen
- `channels.slack.replyToMode`: `off|first|all|batched` (Standardwert `off`)
- `channels.slack.replyToModeByChatType`: pro `direct|group|channel`
- alte Ausweichoption für direkte Chats: `channels.slack.dm.replyToMode`

Manuelle Antwort-Tags werden unterstützt:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

Für explizite Slack-Thread-Antworten aus dem Tool `message` setzen Sie `replyBroadcast: true` zusammen mit `action: "send"` und `threadId` oder `replyTo`, damit Slack die Thread-Antwort zusätzlich im übergeordneten Kanal veröffentlicht. Dies entspricht Slacks Flag `reply_broadcast` von `chat.postMessage` und wird nur für Text- oder Block-Kit-Sendungen unterstützt, nicht für Medien-Uploads.

Wenn ein Aufruf des Tools `message` innerhalb eines Slack-Threads ausgeführt wird und denselben Kanal als Ziel hat, übernimmt OpenClaw normalerweise den aktuellen Slack-Thread gemäß dem effektiven `replyToMode` des Kontos, Chat-Typs oder Kanals. Automatische Antworten und Aufrufe von `send` oder `upload-file` im selben Kanal verwenden dieselbe kanalspezifische Überschreibung. Setzen Sie `topLevel: true` für `action: "send"` oder `action: "upload-file"`, um stattdessen eine neue Nachricht im übergeordneten Kanal zu erzwingen. `threadId: null` wird als gleichwertige Deaktivierung auf oberster Ebene akzeptiert.

<Note>
`replyToMode="off"` deaktiviert optionale ausgehende Slack-Antwort-Threads einschließlich expliziter `[[reply_to_*]]`-Tags. Agent View und Assistant View sind von Slack verwaltete Thread-Erlebnisse. Daher verbleiben ihre Antworten und Status unabhängig von dieser Einstellung an der sichtbaren Wurzel. Andere eingehende Slack-Thread-Sitzungen werden dadurch nicht zusammengeführt. Dies unterscheidet sich von Telegram, wo explizite Tags im Modus `"off"` weiterhin berücksichtigt werden. Slack-Threads verbergen Nachrichten im Kanal, während Telegram-Antworten weiterhin direkt im Verlauf sichtbar bleiben.
</Note>

## Bestätigungsreaktionen

`ackReaction` sendet ein Bestätigungs-Emoji, während OpenClaw eine eingehende Nachricht verarbeitet. `ackReactionScope` bestimmt, _wann_ dieses Emoji tatsächlich gesendet wird.

Standardmäßig bleibt die Bestätigung unverändert, während der native Agenten-/Assistenten-Thread-Status von Slack den Fortschritt mit wechselnden Lademeldungen anzeigt. Setzen Sie `messages.statusReactions.enabled: true`, um stattdessen den Reaktionslebenszyklus für Warteschlange/Denken/Tool/Abschluss/Fehler zu aktivieren.

### Emoji (`ackReaction`)

Auflösungsreihenfolge:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- ersatzweise das Emoji der Agentenidentität (`agents.entries.*.identity.emoji`, andernfalls `"eyes"` / 👀)

Hinweise:

- Slack erwartet Kurzcodes (beispielsweise `"eyes"`).
- Verwenden Sie `""`, um die Reaktion für das Slack-Konto oder global zu deaktivieren.

### Geltungsbereich (`messages.ackReactionScope`)

Der Slack-Provider liest den Geltungsbereich aus `messages.ackReactionScope` (Standardwert `"group-mentions"`). Derzeit gibt es keine Überschreibung auf Slack-Konto- oder Slack-Kanalebene; der Wert gilt global für das Gateway.

Werte:

- `"all"`: in DMs und Gruppen reagieren, einschließlich beiläufiger Raumereignisse.
- `"direct"`: nur in DMs reagieren.
- `"group-all"`: auf jede Gruppennachricht außer beiläufigen Raumereignissen reagieren (keine DMs).
- `"group-mentions"` (Standardwert): in Gruppen reagieren, jedoch nur, wenn der Bot erwähnt wird (oder bei erwähnbaren Gruppen, für die dies aktiviert wurde). **DMs sind ausgeschlossen.**
- `"off"` / `"none"`: niemals reagieren.

<Note>
Der Standardgeltungsbereich (`"group-mentions"`) löst keine Bestätigungsreaktionen in Direktnachrichten oder bei beiläufigen Raumereignissen aus. Um das konfigurierte `ackReaction` (beispielsweise `"eyes"`) bei eingehenden Slack-DMs und stillen Raumereignissen zu sehen, setzen Sie `messages.ackReactionScope` auf `"all"`. `messages.ackReactionScope` wird beim Start des Slack-Providers gelesen. Daher ist ein Neustart des Gateways erforderlich, damit die Änderung wirksam wird.
</Note>

```json5
{
  messages: {
    ackReaction: "eyes",
    ackReactionScope: "all", // in DMs und Gruppen reagieren
  },
}
```

## Text-Streaming

`channels.slack.streaming` steuert das Verhalten der Live-Vorschau:

- `off`: Streaming der Live-Vorschau deaktivieren.
- `partial` (Standardwert): Vorschautext durch die neueste Teilausgabe ersetzen.
- `block`: gestückelte Vorschauaktualisierungen anhängen.
- `progress`: während der Generierung einen Fortschrittsstatustext anzeigen und anschließend den endgültigen Text senden.
- `streaming.preview.toolProgress`: Bei aktiver Entwurfsvorschau werden Tool- und Fortschrittsaktualisierungen in dieselbe bearbeitete Vorschaunachricht geleitet (Standardwert: `true`). Setzen Sie `false`, um separate Tool- und Fortschrittsnachrichten beizubehalten.
- `streaming.preview.commandText` / `streaming.progress.commandText`: auf `status` setzen, um kompakte Tool-Fortschrittszeilen beizubehalten und gleichzeitig unverarbeiteten Befehls-/Ausführungstext auszublenden (Standardwert: `raw`).

Unverarbeiteten Befehls-/Ausführungstext ausblenden und kompakte Fortschrittszeilen beibehalten:

```json
{
  "channels": {
    "slack": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

`channels.slack.streaming.nativeTransport` steuert das native Slack-Text-Streaming, wenn `channels.slack.streaming.mode` den Wert `partial` hat (Standardwert: `true`).

Native Slack-Fortschrittsaufgabenkarten müssen für den Fortschrittsmodus ausdrücklich aktiviert werden. Setzen Sie `channels.slack.streaming.progress.nativeTaskCards` zusammen mit `channels.slack.streaming.mode="progress"` auf `true`, um während der Ausführung eine native Slack-Plan-/Aufgabenkarte zu senden und dieselbe Aufgabenkarte nach Abschluss zu aktualisieren. Ohne dieses Flag behält der Fortschrittsmodus das portable Verhalten der Entwurfsvorschau bei.

- Ein Antwort-Thread muss verfügbar sein, damit natives Text-Streaming und der Slack-Assistenten-Thread-Status angezeigt werden können. Die Thread-Auswahl folgt weiterhin `replyToMode`.
- Kanal-, Gruppenchat- und übergeordnete DM-Stammnachrichten können weiterhin die normale Entwurfsvorschau verwenden, wenn natives Streaming nicht verfügbar ist oder kein Antwort-Thread vorhanden ist.
- Übergeordnete Slack-DMs bleiben standardmäßig außerhalb von Threads und zeigen daher nicht die Thread-artige native Streaming-/Statusvorschau von Slack an; OpenClaw veröffentlicht und bearbeitet stattdessen eine Entwurfsvorschau in der DM.
- Medien und Nicht-Text-Nutzdaten greifen auf die normale Zustellung zurück.
- Abschließende Medien-/Fehlerausgaben brechen ausstehende Vorschauänderungen ab; geeignete abschließende Text-/Blockausgaben werden nur übertragen, wenn sie die Vorschau direkt bearbeiten können.
- Wenn das Streaming während einer Antwort fehlschlägt, greift OpenClaw für die verbleibenden Nutzdaten auf die normale Zustellung zurück.

Entwurfsvorschau anstelle des nativen Slack-Text-Streamings verwenden:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "partial",
        nativeTransport: false,
      },
    },
  },
}
```

Native Slack-Fortschrittsaufgabenkarten aktivieren:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          nativeTaskCards: true,
          render: "rich",
        },
      },
    },
  },
}
```

Veraltete Schlüssel:

- `channels.slack.streamMode` (`replace | status_final | append`) ist ein veralteter Alias für `channels.slack.streaming.mode`.
- Der boolesche Wert `channels.slack.streaming` ist ein veralteter Alias für `channels.slack.streaming.mode` und `channels.slack.streaming.nativeTransport`.
- Die übergeordneten Werte `channels.slack.chunkMode` und `channels.slack.nativeStreaming` sind veraltete Aliasse für `channels.slack.streaming.chunkMode` und `channels.slack.streaming.nativeTransport`.
- Veraltete Aliasse werden zur Laufzeit nicht gelesen; führen Sie `openclaw doctor --fix` aus, um die persistierte Slack-Streaming-Konfiguration mit den kanonischen Schlüsseln neu zu schreiben.

## Ausweichlösung mit Eingabe-Reaktion

`typingReaction` fügt der eingehenden Slack-Nachricht vorübergehend eine Reaktion hinzu, während OpenClaw eine Antwort verarbeitet, und entfernt sie nach Abschluss des Laufs. Dies ist vor allem außerhalb von Thread-Antworten nützlich, die standardmäßig eine Statusanzeige „tippt gerade ...“ verwenden.

Auflösungsreihenfolge:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

Hinweise:

- Slack erwartet Shortcodes (zum Beispiel `"hourglass_flowing_sand"`).
- Die Reaktion erfolgt nach Möglichkeit, und nach Abschluss des Antwort- oder Fehlerpfads wird automatisch versucht, sie zu entfernen.

## Spracheingabe

Um derzeit in Slack mit OpenClaw zu sprechen, senden Sie einen Slack-Audioclip an die OpenClaw-App. Das Diktiermikrofon von Slackbot ist eine separate, Slack-eigene Funktion und keine App-API.

- **[Slackbot-Sprachdiktat](https://slack.com/help/articles/202026038-How-to-use-Slackbot)** befindet sich in der privaten Slackbot-Unterhaltung der Person. Slack wandelt die Aufnahme in einen Slackbot-Prompt um, stellt Drittanbieter-Slack-Apps jedoch über die Events API weder eine Audiodatei noch ein Diktatereignis, einen Prompt oder eine Markierung der Eingabequelle bereit. Das OpenClaw-Slack-Plugin kann diese Funktion weder aktivieren noch empfangen.
- **[Slack-Audio-Clips](https://slack.com/help/articles/4406235165587-Record-audio-and-video-clips-in-Slack)** sind gespeicherte Slack-Dateien, die in einer OpenClaw-DM, einem Kanal oder einem Thread veröffentlicht werden können. OpenClaw lädt einen zugänglichen Clip mit dem Bot-Token herunter, normalisiert die MIME-Metadaten des Slack-Clips und leitet ihn durch die gemeinsame [Audiotranskriptions-Pipeline](/de/nodes/audio). Das empfohlene App-Manifest enthält den erforderlichen Bereich `files:read`.

Audioclips und Slackbot-Diktate haben unterschiedliche Datenschutzmerkmale: Clips unterliegen der Slack-Richtlinie zur Dateiaufbewahrung und werden von OpenClaw zur Transkription heruntergeladen, während Slack angibt, dass Diktataudio nicht gespeichert wird.

In einem Kanal mit `requireMention: true` kann ein Audioclip ohne Beschriftung die Bedingung erfüllen, indem ein konfiguriertes Erwähnungsmuster ausgesprochen wird (`agents.entries.*.groupChat.mentionPatterns`, mit Rückgriff auf `messages.groupChat.mentionPatterns`). OpenClaw autorisiert den Absender, bevor der Clip heruntergeladen oder transkribiert wird, und lässt ihn nur zu, wenn das Transkript übereinstimmt. Ein fehlgeschlagenes oder nicht übereinstimmendes spekulatives Transkript wird zusammen mit dem heruntergeladenen Clip verworfen; es wird nicht im Kanalverlauf aufbewahrt. Die native Slack-Identität `@bot` kann nicht aus Sprache abgeleitet werden. Konfigurieren Sie daher ein gesprochenes Namensmuster oder fügen Sie eine eingegebene Erwähnung hinzu. Wenn die Transkriptwiedergabe aktiviert ist, wird sie erst nach der Zulassung gesendet.

## Medien, Aufteilung und Zustellung

<AccordionGroup>
  <Accordion title="Eingehende Anhänge">
    Slack-Dateianhänge werden von privaten, von Slack gehosteten URLs heruntergeladen (tokenauthentifizierter Anfrageablauf) und bei erfolgreichem Abruf sowie Einhaltung der Größenbeschränkungen im Medienspeicher abgelegt. Dateiplatzhalter enthalten die Slack-Kennung `fileId`, damit Agenten die Originaldatei mit `download-file` abrufen können.

    Downloads verwenden begrenzte Leerlauf- und Gesamtzeitüberschreitungen. Wenn der Abruf einer Slack-Datei stockt oder fehlschlägt, verarbeitet OpenClaw die Nachricht weiter und greift auf den Dateiplatzhalter zurück.

    Die Größenobergrenze für eingehende Daten beträgt zur Laufzeit standardmäßig `20MB`, sofern sie nicht durch `channels.slack.mediaMaxMb` überschrieben wird.

  </Accordion>

  <Accordion title="Ausgehender Text und Dateien">
    - Textabschnitte verwenden `channels.slack.textChunkLimit` (Standardwert `8000`, begrenzt auf die Slack-eigene Nachrichtenlängenbeschränkung)
    - `channels.slack.streaming.chunkMode="newline"` aktiviert die vorrangige Aufteilung nach Absätzen
    - Dateien werden über die Slack-Upload-APIs gesendet und können Thread-Antworten enthalten (`thread_ts`)
    - Bei langen Dateibeschriftungen wird der erste Slack-kompatible Textabschnitt als Upload-Kommentar verwendet; verbleibende Abschnitte werden als Folgenachrichten gesendet
    - Die Obergrenze für ausgehende Medien folgt `channels.slack.mediaMaxMb`, sofern konfiguriert; andernfalls verwenden Kanalübertragungen die MIME-Standardwerte der Medien-Pipeline

  </Accordion>

  <Accordion title="Zustellungsziele">
    Bevorzugte explizite Ziele:

    - `user:<id>` für DMs
    - `channel:<id>` für Kanäle

    Slack-DMs, die nur Text oder Blöcke enthalten, können direkt an Benutzer-IDs gesendet werden; Datei-Uploads und Thread-Übertragungen öffnen die DM zuerst über die Slack-Unterhaltungs-APIs, da diese Pfade eine konkrete Unterhaltungs-ID erfordern.

  </Accordion>
</AccordionGroup>

## Befehle und Slash-Verhalten

Slash-Befehle werden in Slack entweder als einzelner konfigurierter Befehl oder als mehrere native Befehle angezeigt. Konfigurieren Sie `channels.slack.slashCommand`, um die Befehlsstandardwerte zu ändern:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

```txt
/openclaw /help
```

Native Befehle erfordern [zusätzliche Manifest-Einstellungen](#additional-manifest-settings) in Ihrer Slack-App und werden stattdessen mit `channels.slack.commands.native: true` oder `commands.native: true` in globalen Konfigurationen aktiviert.

- Der automatische Modus für native Befehle ist für Slack **deaktiviert**, sodass `commands.native: "auto"` keine nativen Slack-Befehle aktiviert.

```txt
/help
```

Native Argumentmenüs werden in der folgenden Prioritätsreihenfolge dargestellt:

- 3-5 ausreichend kurze Optionen: ein Überlaufmenü („...“)
- mehr als 100 Optionen bei verfügbarer asynchroner Optionsfilterung: externe Auswahl
- 1-2 Optionen oder eine Option, deren codierter Wert für eine Auswahl zu lang ist: Schaltflächenblöcke
- andernfalls (6-100 Optionen oder mehr als 100 ohne asynchrone Filterung): statisches Auswahlmenü, aufgeteilt in jeweils 100 Optionen pro Menü

```txt
/think
```

Slash-Sitzungen verwenden isolierte Schlüssel wie `agent:<agentId>:slack:slash:<userId>` und leiten Befehlsausführungen über `CommandTargetSessionKey` weiterhin an die Sitzung der Zielunterhaltung weiter.

## Native Diagramme

Der öffentliche [`data_visualization`-Block von Block Kit](https://docs.slack.dev/reference/block-kit/blocks/data-visualization-block/)
von Slack stellt Linien-, Balken-, Flächen- und Kreisdiagramme in Nachrichten dar. OpenClaw ordnet den portablen
`presentation`-Block `chart` dieser nativen Form zu; zusätzlich zum normalen
Nachrichtenzugriff `chat:write` sind weder ein zusätzlicher OAuth-Bereich noch ein
Datei-Upload, Bildrenderer oder eine Slack-Konfiguration erforderlich.

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "bar",
      "title": "Quartalsumsatz",
      "categories": ["Q1", "Q2"],
      "series": [{ "name": "Umsatz", "values": [120, 145] }],
      "xLabel": "Quartal"
    }
  ]
}
```

Die Slack-Beschränkungen werden vor der nativen Darstellung durchgesetzt:

- Titel und optionale Achsenbeschriftungen: 50 Zeichen
- Kreisdiagramm: 1-12 positive Segmente
- Linien-/Balken-/Flächendiagramm: 1-12 eindeutig benannte Datenreihen und 1-20 gemeinsame Kategorien
- Segment-, Kategorie- und Datenreihenbeschriftungen: 20 Zeichen
- Jede Datenreihe muss für jede Kategorie einen endlichen Wert enthalten; Werte außerhalb von Kreisdiagrammen
  dürfen negativ sein

Jedes native Diagramm enthält außerdem eine übergeordnete Textdarstellung für
Screenreader, Benachrichtigungen, die Sitzungsspiegelung und Clients, die den
Block nicht darstellen können. Bei standardmäßigen Präsentationsübertragungen an andere OpenClaw-Kanäle
werden dieselben deterministischen Diagrammdaten als Text gesendet, sofern diese
keine native Diagrammunterstützung ausweisen. Wenn Slack das Diagramm während einer schrittweisen Einführung mit
`invalid_blocks` ablehnt, entfernt OpenClaw die abgelehnten nativen Datenblöcke,
behält vorhandene gleichgeordnete Steuerelemente bei und sendet die vollständige
Diagrammdarstellung als sichtbaren Text.

Slack akzeptiert derzeit bis zu zwei `data_visualization`-Blöcke pro Nachricht. Wenn
eine Präsentation mehr als zwei gültige Diagramme enthält, behält OpenClaw deren Reihenfolge
bei und setzt die native Darstellung in Folgenachrichten mit jeweils höchstens zwei
Diagrammen fort.

Die [Entwicklerveröffentlichung](https://docs.slack.dev/changelog/2026/06/16/block-kit-data-visualization-block/)
von Slack dokumentiert den Block als App-seitige Block-Kit-Funktion und nennt keine Einschränkung
auf kostenpflichtige Tarife. Die Angaben zur Berechtigung für Business+/Enterprise gelten für
die automatische KI-Diagrammerstellung von Slackbot, die davon getrennt ist, dass eine App
ein bereits strukturiertes Block-Kit-Diagramm sendet. Diagramme sind reine Nachrichtenblöcke und keine
Inhalte für App Home, Modalfenster oder Canvas.

## Native Tabellen

Der aktuelle [`data_table`-Block von Block Kit](https://docs.slack.dev/reference/block-kit/blocks/data-table-block/)
von Slack stellt strukturierte Zeilen und Spalten in Nachrichten dar. OpenClaw ordnet einen expliziten
portablen `presentation`-Block `table` dem Wert `data_table` zu; der veraltete
[`table`-Block](https://docs.slack.dev/reference/block-kit/blocks/table-block/)
von Slack wird nicht verwendet. Zusätzlich zum normalen Nachrichtenzugriff
`chat:write` sind weder ein zusätzlicher OAuth-Bereich noch eine Slack-Konfiguration erforderlich.

```json
{
  "blocks": [
    {
      "type": "table",
      "caption": "Offene Pipeline",
      "headers": ["Konto", "Phase", "ARR"],
      "rows": [
        ["Acme", "Gewonnen", 125000],
        ["Globex", "Prüfung", 82000]
      ],
      "rowHeaderColumnIndex": 0
    }
  ]
}
```

OpenClaw ordnet Kopfzeilen- und Zeichenfolgenzellen Slack-Zellen vom Typ `raw_text` zu. Numerische Zellen
werden `raw_number` zugeordnet, wobei der endliche numerische Wert für natives Sortieren
und Filtern erhalten bleibt. `rowHeaderColumnIndex` kennzeichnet, sofern vorhanden, diese nullbasierte
Spalte als Slack-Zeilenüberschriften.

Die veröffentlichten `data_table`-Beschränkungen von Slack werden vor der nativen Darstellung durchgesetzt:

- 1-20 Spalten
- 1-100 Datenzeilen zuzüglich der Kopfzeile
- dieselbe Anzahl von Zellen in jeder Zeile
- höchstens insgesamt 10.000 Zeichen in allen Tabellenzellen einer Nachricht

Mehrere gültige Tabellenblöcke können nativ dargestellt werden, solange die Nachricht
innerhalb der Gesamtzeichenbegrenzung bleibt. Eine Tabelle, die nicht innerhalb der
nativen Beschränkungen dargestellt werden kann, wird stattdessen als vollständiger deterministischer Text ausgegeben, ohne Zeilen oder
Zellen zu verlieren. Wenn dieser Text die Länge einer Slack-Nachricht überschreitet, verwenden Übertragungen und Slash-Antworten
geordnete Textabschnitte. Tabellenänderungen schlagen mit einem ausdrücklichen Größenfehler fehl, statt
Zeilen einer vorhandenen Nachricht unbemerkt abzuschneiden.

Jede native Tabelle, die aus einer portablen Darstellung erzeugt wird, enthält außerdem eine übergeordnete
Textdarstellung für Screenreader, Benachrichtigungen, Sitzungsspiegelung und
Clients, die den Block nicht darstellen können. Unverarbeitete Diagramm- und Tabellenwerte bleiben
im Fallback unverändert, sodass Zelldaten wie `<@U123>` nicht zu einer Slack-Erwähnung werden.
Wenn Slack native Diagramm- oder Tabellenblöcke mit `invalid_blocks` ablehnt, entfernt OpenClaw
alle nativen Datenblöcke in einem einzigen begrenzten Wiederherstellungsschritt, behält gültige
benachbarte Blöcke wie Schaltflächen und Auswahlfelder bei und sendet den vollständigen sichtbaren Diagramm-
und Tabellentext mit deaktivierter Slack-Formatierung. Die Zustellung von Slash-Befehlen
verfolgt Slacks Budget von fünf Aufrufen für `response_url` über den gesamten Befehl hinweg. Vor jedem
Antwortstapel wählt sie einen vollständigen Plan aus, der in die verbleibenden Aufrufe passt, oder schlägt fehl,
bevor dieser Stapel veröffentlicht wird.

Nur explizite `presentation`-Tabellenblöcke werden zu nativen Tabellen hochgestuft.
Markdown-Pipe-Tabellen bleiben verfasster Text; OpenClaw versucht nicht, die Tabellenstruktur
oder Zelltypen zu erraten. Bestehende vertrauenswürdige Slack-native Erzeuger können weiterhin
unverarbeitete Blöcke über `channelData.slack.blocks` durchreichen; OpenClaw leitet Fallback-
Text aus gültigen unverarbeiteten `data_table`-Zellen ab, während fehlerhafte benutzerdefinierte Blöcke
auf ihre Beschriftung oder den allgemeinen Block-Kit-Fallback zurückfallen können. Portable Ausgaben von Agenten, CLI
und Plugins sollten `presentation` verwenden.

## Interaktive Antworten

Slack kann von Agenten erstellte interaktive Antwortsteuerelemente darstellen, diese Funktion ist jedoch standardmäßig deaktiviert.
Für neue Ausgaben von Agenten, CLI und Plugins sind die gemeinsamen
`presentation`-Schaltflächen oder Auswahlblöcke vorzuziehen. Sie verwenden denselben Slack-Interaktionspfad
und können zugleich auf anderen Kanälen auf eine einfachere Darstellung zurückfallen.

Global aktivieren:

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

Oder nur für ein Slack-Konto aktivieren:

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

Wenn die Funktion aktiviert ist, können Agenten weiterhin veraltete, ausschließlich für Slack bestimmte Antwortdirektiven ausgeben:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

Diese Direktiven werden in Slack Block Kit kompiliert und leiten Klicks oder Auswahlen
über den bestehenden Ereignispfad für Slack-Interaktionen zurück. Behalten Sie sie für alte
Prompts und Slack-spezifische Ausweichmöglichkeiten bei; verwenden Sie für neue
portable Steuerelemente die gemeinsame Darstellung.

Die APIs des Direktiven-Compilers sind für neuen Erzeugercode ebenfalls veraltet:

- `compileSlackInteractiveReplies(...)`
- `parseSlackOptionsLine(...)`
- `isSlackInteractiveRepliesEnabled(...)`
- `buildSlackInteractiveBlocks(...)`

Verwenden Sie `presentation`-Payloads und `buildSlackPresentationBlocks(...)` für neue
in Slack dargestellte Steuerelemente.

Hinweise:

- Dies ist eine Slack-spezifische Legacy-Benutzeroberfläche. Andere Kanäle übersetzen Slack-Block-
  Kit-Direktiven nicht in ihre eigenen Schaltflächensysteme.
- Die interaktiven Callback-Werte sind von OpenClaw erzeugte undurchsichtige Token, keine unverarbeiteten, vom Agenten erstellten Werte.
- Wenn erzeugte interaktive Blöcke die Slack-Block-Kit-Grenzwerte überschreiten würden, greift OpenClaw auf die ursprüngliche Textantwort zurück, statt einen ungültigen Block-Payload zu senden.

### Plugin-eigene Modal-Übermittlungen

Slack-Plugins, die einen interaktiven Handler registrieren, können außerdem modale
`view_submission`- und `view_closed`-Lebenszyklusereignisse empfangen, bevor OpenClaw
den Payload für das für den Agenten sichtbare Systemereignis komprimiert. Verwenden Sie beim Öffnen
eines Slack-Modals eines dieser Routing-Muster:

- Setzen Sie `callback_id` auf `openclaw:<namespace>:<payload>`.
- Oder behalten Sie ein vorhandenes `callback_id` bei und fügen Sie `pluginInteractiveData:
"<namespace>:<payload>"` in das modale `private_metadata` ein.

Der Handler empfängt `ctx.interaction.kind` als `view_submission` oder
`view_closed`, normalisiertes `inputs` und das vollständige unverarbeitete `stateValues`-Objekt von
Slack. Ein ausschließlich auf der Callback-ID basierendes Routing genügt, um den Plugin-Handler aufzurufen; schließen Sie
die vorhandenen Benutzer-/Sitzungs-Routingfelder des Modals `private_metadata` ein, wenn das
Modal zusätzlich ein für den Agenten sichtbares Systemereignis erzeugen soll. Der Agent empfängt ein
kompaktes, geschwärztes `Slack interaction: ...`-Systemereignis. Wenn der Handler
`systemEvent.summary`, `systemEvent.reference` oder `systemEvent.data` zurückgibt, werden diese
Felder in dieses kompakte Ereignis aufgenommen, sodass der Agent auf
Plugin-eigenen Speicher verweisen kann, ohne den vollständigen Formular-Payload zu sehen.

## Native Genehmigungen in Slack

Slack kann mit interaktiven Schaltflächen und Interaktionen als nativer Genehmigungsclient fungieren, statt auf die Web-Benutzeroberfläche oder das Terminal zurückzugreifen.

- Exec- und Plugin-Genehmigungen können als Slack-native Block-Kit-Aufforderungen dargestellt werden.
- `channels.slack.execApprovals.*` bleibt die Konfiguration zur Aktivierung des nativen Clients für Exec-Genehmigungen sowie für das DM-/Kanal-Routing.
- DMs für Exec-Genehmigungen verwenden `channels.slack.execApprovals.approvers` oder `commands.ownerAllowFrom`.
- Plugin-Genehmigungen verwenden Slack-native Schaltflächen, wenn Slack für die Ursprungssitzung als nativer Genehmigungsclient aktiviert ist oder wenn `approvals.plugin` zur ursprünglichen Slack-Sitzung oder zu einem Slack-Ziel routet.
- DMs für Plugin-Genehmigungen verwenden Slack-Plugin-Genehmiger aus `channels.slack.allowFrom`, das benannte Konto `allowFrom` oder die Standardroute des Kontos.
- Die Autorisierung der Genehmiger wird weiterhin durchgesetzt: Genehmiger, die ausschließlich für Exec zuständig sind, können Plugin-Anfragen nur genehmigen, wenn sie auch Plugin-Genehmiger sind.

Hierbei wird dieselbe gemeinsame Oberfläche für Genehmigungsschaltflächen wie bei anderen Kanälen verwendet. Wenn `interactivity` in den Einstellungen Ihrer Slack-App aktiviert ist, werden Genehmigungsaufforderungen direkt in der Unterhaltung als Block-Kit-Schaltflächen dargestellt.
Wenn diese Schaltflächen vorhanden sind, bilden sie die primäre Genehmigungsoberfläche; OpenClaw
sollte einen manuellen `/approve`-Befehl nur einschließen, wenn das Werkzeugergebnis angibt, dass Chat-
Genehmigungen nicht verfügbar sind oder die manuelle Genehmigung der einzige Weg ist.

Konfigurationspfad:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (optional; greift nach Möglichkeit auf `commands.ownerAllowFrom` zurück)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`, Standard: `dm`)
- `agentFilter`, `sessionFilter`

Slack aktiviert native Exec-Genehmigungen automatisch, wenn `enabled` nicht gesetzt oder `"auto"` ist und mindestens ein
Exec-Genehmiger aufgelöst wird. Slack kann über diesen nativen Clientpfad auch native Plugin-Genehmigungen verarbeiten,
wenn Slack-Plugin-Genehmiger aufgelöst werden und die Anfrage den Filtern des nativen Clients entspricht. Setzen Sie
`enabled: false`, um Slack ausdrücklich als nativen Genehmigungsclient zu deaktivieren. Setzen Sie `enabled: true`,
um native Genehmigungen zu erzwingen, wenn Genehmiger aufgelöst werden. Das Deaktivieren von Slack-Exec-Genehmigungen deaktiviert nicht
die native Zustellung von Slack-Plugin-Genehmigungen, die über `approvals.plugin` aktiviert ist; für die Zustellung von Plugin-Genehmigungen
werden stattdessen Slack-Plugin-Genehmiger verwendet.

Standardverhalten ohne explizite Konfiguration für Slack-Exec-Genehmigungen:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

Eine explizite Slack-native Konfiguration ist nur erforderlich, wenn Sie Genehmiger überschreiben, Filter hinzufügen oder
die Zustellung im Ursprungs-Chat aktivieren möchten:

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

Die gemeinsame `approvals.exec`-Weiterleitung ist davon getrennt. Verwenden Sie sie nur, wenn Aufforderungen zur Exec-Genehmigung zusätzlich
an andere Chats oder explizite Out-of-Band-Ziele weitergeleitet werden müssen. Die gemeinsame `approvals.plugin`-Weiterleitung ist ebenfalls
getrennt; die native Slack-Zustellung unterdrückt diesen Fallback nur, wenn Slack die Plugin-
Genehmigungsanfrage nativ verarbeiten kann.

`/approve` im selben Chat funktioniert ebenfalls in Slack-Kanälen und DMs, die bereits Befehle unterstützen. Das vollständige Weiterleitungsmodell für Genehmigungen finden Sie unter [Exec-Genehmigungen](/de/tools/exec-approvals).

## Ereignisse und Betriebsverhalten

- Bearbeitungen und Löschungen von Nachrichten werden auf Systemereignisse abgebildet.
- Thread-Übertragungen (Thread-Antworten mit „Also send to channel“) werden als normale Benutzernachrichten verarbeitet.
- Ereignisse zum Hinzufügen und Entfernen von Reaktionen werden auf Systemereignisse abgebildet.
- Beitritte und Austritte von Mitgliedern, erstellte und umbenannte Kanäle sowie Ereignisse zum Hinzufügen und Entfernen von Pins werden auf Systemereignisse abgebildet.
- Optionales Präsenz-Polling kann den beobachteten Übergang eines menschlichen Teilnehmers von `away` zu `active` der zuletzt aktiven, geeigneten Slack-Sitzung dieses Teilnehmers zuordnen. Standardmäßig ist es deaktiviert.
- `channel_id_changed` kann Kanalkonfigurationsschlüssel migrieren, wenn `configWrites` aktiviert ist.
- Metadaten zu Kanalthema und -zweck werden als nicht vertrauenswürdiger Kontext behandelt und können in den Routing-Kontext eingefügt werden.
- Agent-View-Entitäten `app_context` werden in der Slack-Relevanzreihenfolge validiert und ausschließlich als strukturierter, nicht vertrauenswürdiger Kontext offengelegt; ein ausgelassener Kontext löscht die Daten für den aktuellen Turn, statt veraltete Entitäten wiederzuverwenden.
- Der Thread-Starter und die anfängliche Kontextübernahme aus dem Thread-Verlauf werden gegebenenfalls anhand konfigurierter Absender-Zulassungslisten gefiltert.
- Blockaktionen, Kurzbefehle und modale Interaktionen erzeugen strukturierte `Slack interaction: ...`-Systemereignisse mit umfangreichen Payload-Feldern:
  - Blockaktionen: ausgewählte Werte, Beschriftungen, Auswahlwerte und `workflow_*`-Metadaten
  - globale Kurzbefehle: Callback- und Akteursmetadaten, geroutet an die direkte Sitzung des Akteurs
  - Nachrichtenkurzbefehle: Callback-, Akteurs-, Kanal-, Thread- und Kontextdaten der ausgewählten Nachricht
  - modale `view_submission`- und `view_closed`-Ereignisse mit gerouteten Kanalmetadaten und Formulareingaben

Definieren Sie globale oder Nachrichtenkurzbefehle in der Konfiguration Ihrer Slack-App und verwenden Sie eine beliebige nicht leere Callback-ID. OpenClaw bestätigt passende Kurzbefehl-Payloads, wendet dieselben Absenderrichtlinien für DMs und Kanäle wie bei anderen Slack-Interaktionen an und stellt das bereinigte Ereignis für die geroutete Agentensitzung in die Warteschlange. Trigger-IDs und Antwort-URLs werden im Agentenkontext geschwärzt.

### Präsenzereignisse

Slack sendet Präsenzänderungen weder über die Events API noch über Socket Mode. OpenClaw kann stattdessen [`users.getPresence`](https://docs.slack.dev/reference/methods/users.getPresence/) für menschliche Teilnehmer abfragen, deren Nachrichten die normalen Slack-Zugriffs- und Routing-Prüfungen bestanden haben.

```json5
{
  channels: {
    slack: {
      presenceEvents: { mode: "auto" },
      channels: {
        C0123456789: { presenceEvents: { mode: "on" } },
        C0987654321: { presenceEvents: { mode: "off" } },
      },
    },
  },
}
```

- `off` (Standard): kein Präsenz-Timer und keine Slack-API-Aufrufe.
- `auto`: überwacht DMs, MPIMs und Slack-Threads, die in den letzten 24 Stunden aktiv waren, mit höchstens 8 beobachteten menschlichen Teilnehmern. Kanalsitzungen auf oberster Ebene sind ausgeschlossen.
- `on`: überwacht dieselben Unterhaltungen ohne Teilnehmerbegrenzung und schließt Kanalsitzungen auf oberster Ebene ein. Verwenden Sie eine kanalspezifische Überschreibung, um einen Kanal zu erzwingen oder zu unterdrücken.

OpenClaw fragt pro Slack-Konto höchstens 45 eindeutige Benutzer pro Minute ab, übernimmt das erste Ergebnis, ohne den Agenten zu aktivieren, und aktiviert ihn nur bei einem beobachteten Übergang von `away` zu `active`. Pro Slack-Konto und Benutzer gilt eine dauerhafte Abklingzeit von 8 Stunden, selbst wenn diese Person an mehreren Threads teilnimmt. Das Ereignis wird ausschließlich an die zuletzt aktive, geeignete Unterhaltung dieser Person geroutet und weist den Agenten an, den Speicher/das Wiki und den bekannten Zeitzonenkontext zu konsultieren, bevor er entscheidet, ob er eine kurze Begrüßung sendet. Der Agent kann stumm bleiben.

Das Bot-Token benötigt `users:read`, das bereits im empfohlenen Manifest enthalten ist. Präsenzereignisse sind für organisationsweite Enterprise-Grid-Installationen nicht verfügbar.

## Konfigurationsreferenz

Primäre Referenz: [Konfigurationsreferenz – Slack](/de/gateway/config-channels#slack).

<Accordion title="Wichtige Slack-Felder">

- Modus/Authentifizierung: `identity`, `mode`, `enterpriseOrgInstall`, `botToken`, `appToken`, `userToken`, `signingSecret`, `webhookPath`, `accounts.*`
- DM-Zugriff: `dm.enabled`, `dmPolicy`, `allowFrom` (veraltet: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
- Kompatibilitätsschalter: `dangerouslyAllowNameMatching` (Notfalloption; deaktiviert lassen, sofern nicht benötigt)
- Kanalzugriff: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`, `implicitMentions.*`
- Threads/Verlauf: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- Aktivierung durch Anwesenheit: `presenceEvents.mode`, `channels.*.presenceEvents.mode` (`off|auto|on`; Standardwert `off`)
- Zustellung: `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `streaming`, `streaming.nativeTransport`, `streaming.preview.toolProgress`
- Vorschauen: `unfurlLinks` (Standardwert: `false`), `unfurlMedia` zur Steuerung der Link-/Medienvorschau für `chat.postMessage`; setzen Sie `unfurlLinks: true`, um Linkvorschauen wieder zu aktivieren
- Betrieb/Funktionen: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

</Accordion>

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Keine Antworten in Kanälen">
    Prüfen Sie der Reihe nach:

    - `groupPolicy`
    - Kanal-Zulassungsliste (`channels.slack.channels`) — **Schlüssel müssen Kanal-IDs sein** (`C12345678`), keine Namen (`#channel-name`). Namensbasierte Schlüssel schlagen unter `groupPolicy: "allowlist"` stillschweigend fehl, da das Kanal-Routing standardmäßig vorrangig anhand der ID erfolgt. So finden Sie eine ID: Klicken Sie in Slack mit der rechten Maustaste auf den Kanal → **Copy link** — der Wert `C...` am Ende der URL ist die Kanal-ID.
    - `requireMention`
    - kanalspezifische `users`-Zulassungsliste
    - `messages.groupChat.visibleReplies`: Normale Gruppen-/Kanalanfragen verwenden standardmäßig `"automatic"`. Wenn Sie `"message_tool"` aktiviert haben und die Protokolle Assistententext ohne Aufruf von `message(action=send)` zeigen, hat das Modell den sichtbaren Pfad des Nachrichten-Tools nicht verwendet. Der endgültige Text bleibt in diesem Modus privat; prüfen Sie das ausführliche Gateway-Protokoll auf Metadaten unterdrückter Nutzdaten oder setzen Sie die Option auf `"automatic"`, wenn jede normale abschließende Assistentenantwort über den veralteten Pfad veröffentlicht werden soll.
    - `messages.groupChat.unmentionedInbound`: Wenn der Wert `"room_event"` ist, dient nicht erwähnte Kommunikation in zugelassenen Kanälen als Umgebungskontext und bleibt stumm, sofern der Agent nicht das Tool `message` aufruft. Siehe [Ereignisse in Umgebungsräumen](/de/channels/ambient-room-events).

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

    Nützliche Befehle:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="DM-Nachrichten werden ignoriert">
    Prüfen Sie:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (oder veraltet `channels.slack.dm.policy`)
    - Kopplungsgenehmigungen/Zulassungslisteneinträge (`dmPolicy: "open"` erfordert weiterhin `channels.slack.allowFrom: ["*"]`)
    - Gruppen-DMs verwenden die MPIM-Verarbeitung; aktivieren Sie `channels.slack.dm.groupEnabled` und nehmen Sie, sofern konfiguriert, die MPIM in `channels.slack.dm.groupChannels` auf
    - DM-Ereignisse des Slack Assistant: Ausführliche Protokolle, die `drop message_changed` erwähnen,
      bedeuten normalerweise, dass Slack ein bearbeitetes Assistant-Thread-Ereignis gesendet hat, ohne dass
      in den Nachrichtenmetadaten ein menschlicher Absender ermittelt werden konnte

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Socket Mode stellt keine Verbindung her">
    Überprüfen Sie Bot- und App-Tokens sowie die Aktivierung von Socket Mode in den Slack-App-Einstellungen.
    Das App-Level Token benötigt `connections:write`, und das Bot User OAuth Token
    muss zu derselben Slack-App und demselben Workspace gehören wie das App-Token.

    Wenn `openclaw channels status --probe --json` den Wert `botTokenStatus` oder
    `appTokenStatus: "configured_unavailable"` anzeigt, ist das Slack-Konto
    konfiguriert, aber die aktuelle Laufzeit konnte den durch SecretRef referenzierten
    Wert nicht auflösen.

    Protokolle wie `slack socket mode failed to start; retry ...` kennzeichnen behebbare
    Startfehler. Fehlende Berechtigungsbereiche, widerrufene Tokens und ungültige Authentifizierung führen
    stattdessen sofort zum Abbruch. Ein Protokolleintrag `slack token mismatch ...` bedeutet, dass Bot-Token und App-Token
    offenbar zu unterschiedlichen Slack-Apps gehören; korrigieren Sie die Anmeldedaten der Slack-App.

  </Accordion>

  <Accordion title="HTTP-Modus empfängt keine Ereignisse">
    Überprüfen Sie:

    - Signaturgeheimnis
    - Webhook-Pfad
    - Slack Request URLs (Events + Interactivity + Slash Commands)
    - eindeutiger Wert für `webhookPath` pro HTTP-Konto
    - die öffentliche URL beendet TLS und leitet Anfragen an den Gateway-Pfad weiter
    - der Pfad `request_url` der Slack-App stimmt exakt mit `channels.slack.webhookPath` überein (Standardwert `/slack/events`)

    Wenn `signingSecretStatus: "configured_unavailable"` in Konto-
    Snapshots erscheint, ist das HTTP-Konto konfiguriert, aber die aktuelle Laufzeit konnte
    das durch SecretRef referenzierte Signaturgeheimnis nicht auflösen.

    Ein wiederholt auftretender Protokolleintrag `slack: webhook path ... already registered` bedeutet, dass zwei HTTP-
    Konten denselben Wert für `webhookPath` verwenden; weisen Sie jedem Konto einen eigenen Pfad zu.

  </Accordion>

  <Accordion title="Native Befehle/Slash-Befehle werden nicht ausgeführt">
    Überprüfen Sie, ob Folgendes beabsichtigt war:

    - Modus für native Befehle (`channels.slack.commands.native: true`) mit entsprechenden in Slack registrierten Slash-Befehlen
    - oder der Modus für einen einzelnen Slash-Befehl (`channels.slack.slashCommand.enabled: true`)

    Slack erstellt oder entfernt Slash-Befehle nicht automatisch. `commands.native: "auto"` aktiviert keine nativen Slack-Befehle; verwenden Sie `true` und erstellen Sie die entsprechenden Befehle in der Slack-App. Im HTTP-Modus muss jeder Slack-Slash-Befehl die Gateway-URL enthalten. Im Socket Mode gehen Befehlsnutzdaten über den WebSocket ein und Slack ignoriert `slash_commands[].url`.

    Prüfen Sie außerdem `commands.useAccessGroups`, die DM-Autorisierung, Kanal-Zulassungslisten
    und kanalspezifische `users`-Zulassungslisten. Slack gibt für
    blockierte Absender von Slash-Befehlen kurzlebige Fehler zurück, darunter:

    - `This channel is not allowed.`
    - `You are not authorized to use this command here.`

  </Accordion>
</AccordionGroup>

## Referenz für Anhangsmedien

Slack kann heruntergeladene Medien an den Agentendurchlauf anhängen, wenn das Herunterladen von Slack-Dateien erfolgreich ist und die Größenbeschränkungen eingehalten werden. Audioclips können transkribiert werden, Bilddateien können den Pfad zur Medienerkennung durchlaufen oder direkt an ein für Bildverarbeitung geeignetes Antwortmodell übergeben werden, und andere Dateien bleiben als herunterladbarer Dateikontext verfügbar.

### Unterstützte Medientypen

| Medientyp                     | Quelle               | Aktuelles Verhalten                                                                  | Hinweise                                                                     |
| ------------------------------ | -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Slack-Audioclips              | Slack-Datei-URL       | Heruntergeladen und durch die gemeinsame Audiotranskription geleitet                          | Erfordert `files:read` und ein funktionierendes `tools.media.audio`-Modell oder eine funktionierende CLI      |
| JPEG-/PNG-/GIF-/WebP-Bilder | Slack-Datei-URL       | Heruntergeladen und für die bildverarbeitungsfähige Verarbeitung an den Durchlauf angehängt                   | Höchstgrenze pro Datei: `channels.slack.mediaMaxMb` (Standardwert 20 MB)                 |
| PDF-Dateien                      | Slack-Datei-URL       | Heruntergeladen und als Dateikontext für Tools wie `download-file` oder `pdf` bereitgestellt | Eingehende Slack-Verarbeitung konvertiert PDFs nicht automatisch in Eingaben für die Bildverarbeitung |
| Andere Dateien                    | Slack-Datei-URL       | Wenn möglich heruntergeladen und als Dateikontext bereitgestellt                              | Binärdateien werden nicht als Bildeingaben behandelt                               |
| Thread-Antworten                 | Dateien der Thread-Ausgangsnachricht | Dateien der Ausgangsnachricht können als Kontext geladen werden, wenn die Antwort keine direkten Medien enthält  | Ausgangsnachrichten, die nur Dateien enthalten, verwenden einen Anhangsplatzhalter                          |
| Nachrichten mit mehreren Dateien            | Mehrere Slack-Dateien | Jede Datei wird unabhängig ausgewertet                                              | Die Slack-Verarbeitung ist auf acht Dateien pro Nachricht begrenzt                     |

### Eingehende Verarbeitungspipeline

Wenn eine Slack-Nachricht mit Dateianhängen eingeht:

1. OpenClaw lädt die Datei mit dem Bot-Token von der privaten Slack-URL herunter.
2. Nach erfolgreichem Herunterladen wird die Datei in den Medienspeicher geschrieben.
3. Die Pfade und Inhaltstypen der heruntergeladenen Medien werden dem eingehenden Kontext hinzugefügt.
4. Audioclips werden an die gemeinsame Transkriptionspipeline weitergeleitet; bildfähige Modell-/Tool-Pfade können Bildanhänge aus demselben Kontext verwenden.
5. Andere Dateien bleiben für Tools, die sie verarbeiten können, als Dateimetadaten oder Medienreferenzen verfügbar.

### Vererbung von Anhängen der Thread-Ausgangsnachricht

Wenn eine Nachricht in einem Thread eingeht (also ein übergeordnetes Element `thread_ts` besitzt):

- Wenn die Antwort selbst keine direkten Medien enthält und die eingebundene Ausgangsnachricht Dateien besitzt, kann Slack die Ausgangsdateien als Kontext der Thread-Ausgangsnachricht laden.
- Ausgangsdateien werden nur beim Initialisieren einer neuen oder zurückgesetzten Thread-Sitzung geladen. Spätere Antworten, die nur Text enthalten, verwenden den vorhandenen Sitzungskontext erneut und hängen die Ausgangsdateien nicht erneut als neue Medien an.
- Direkte Antwortanhänge haben Vorrang vor Anhängen der Ausgangsnachricht.
- Eine Ausgangsnachricht, die nur Dateien und keinen Text enthält, wird durch einen Anhangsplatzhalter dargestellt, damit der Rückfallpfad ihre Dateien weiterhin einbeziehen kann.

### Verarbeitung mehrerer Anhänge

Wenn eine einzelne Slack-Nachricht mehrere Dateianhänge enthält:

- Jeder Anhang wird unabhängig durch die Medienpipeline verarbeitet.
- Referenzen auf heruntergeladene Medien werden im Nachrichtenkontext zusammengeführt.
- Die Verarbeitungsreihenfolge entspricht der Dateireihenfolge von Slack in den Ereignisnutzdaten.
- Ein Fehler beim Herunterladen eines Anhangs blockiert die anderen Anhänge nicht.

### Größen-, Download- und Modellbeschränkungen

- **Größenbeschränkung**: Standardmäßig 20 MB pro Datei. Konfigurierbar über `channels.slack.mediaMaxMb`.
- **Höchstgrenze für die Audiotranskription**: Der Wert `maxBytes` des ausgewählten audiofähigen Eintrags `tools.media.models[]` gilt ebenfalls, wenn die heruntergeladene Datei an einen Transkriptions-Provider oder eine CLI gesendet wird.
- **Downloadfehler**: Dateien, die Slack nicht bereitstellen kann, abgelaufene URLs, nicht zugängliche Dateien, zu große Dateien sowie HTML-Antworten für Slack-Authentifizierung/-Anmeldung werden übersprungen, statt als nicht unterstützte Formate gemeldet zu werden.
- **Bildverarbeitungsmodell**: Die Bildanalyse verwendet das aktive Antwortmodell, wenn es Bildverarbeitung unterstützt, oder das unter `agents.defaults.imageModel` konfigurierte Bildmodell.

### Bekannte Einschränkungen

| Szenario                                      | Aktuelles Verhalten                                                                   | Problemumgehung                                                                    |
| --------------------------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| Abgelaufene Slack-Datei-URL                   | Datei wird übersprungen; es wird kein Fehler angezeigt                               | Laden Sie die Datei erneut in Slack hoch                                           |
| Audiotranskription nicht verfügbar            | Der Clip bleibt angehängt, aber es wird kein Transkript erstellt                     | Konfigurieren Sie `tools.media.audio` oder installieren Sie eine unterstützte lokale Transkriptions-CLI |
| Clip ohne Bildunterschrift passiert kein Erwähnungs-Gate | Wird nach privater spekulativer Transkription verworfen; Transkript und Download werden gelöscht | Konfigurieren Sie ein Erwähnungsmuster für gesprochene Namen, fügen Sie eine getippte Bot-Erwähnung hinzu oder verwenden Sie eine DM |
| Vision-Modell nicht konfiguriert              | Bildanhänge werden als Medienreferenzen gespeichert, aber nicht als Bilder analysiert | Konfigurieren Sie `agents.defaults.imageModel` oder verwenden Sie ein visionsfähiges Antwortmodell |
| Sehr große Bilder (standardmäßig > 20 MB)     | Werden gemäß Größenlimit übersprungen                                                | Erhöhen Sie `channels.slack.mediaMaxMb`, falls Slack dies zulässt                           |
| Weitergeleitete/geteilte Anhänge              | Text sowie von Slack gehostete Bild-/Dateimedien werden nach bestem Bemühen verarbeitet | Teilen Sie sie direkt im OpenClaw-Thread erneut                                    |
| PDF-Anhänge                                   | Werden als Datei-/Medienkontext gespeichert und nicht automatisch durch die Bilderkennung geleitet | Verwenden Sie `download-file` für Dateimetadaten oder das Tool `pdf` zur PDF-Analyse |

### Zugehörige Dokumentation

- [Pipeline zum Medienverständnis](/de/nodes/media-understanding)
- [Audio- und Sprachnotizen](/de/nodes/audio)
- [PDF-Tool](/de/tools/pdf)

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Kopplung" icon="link" href="/de/channels/pairing">
    Koppeln Sie einen Slack-Benutzer mit dem Gateway.
  </Card>
  <Card title="Gruppen" icon="users" href="/de/channels/groups">
    Verhalten von Kanälen und Gruppen-DMs.
  </Card>
  <Card title="Kanalrouting" icon="route" href="/de/channels/channel-routing">
    Leiten Sie eingehende Nachrichten an Agenten weiter.
  </Card>
  <Card title="Sicherheit" icon="shield" href="/de/gateway/security">
    Bedrohungsmodell und Absicherung.
  </Card>
  <Card title="Konfiguration" icon="sliders" href="/de/gateway/configuration">
    Konfigurationsstruktur und Prioritätsreihenfolge.
  </Card>
  <Card title="Slash-Befehle" icon="terminal" href="/de/tools/slash-commands">
    Befehlskatalog und Verhalten.
  </Card>
</CardGroup>
