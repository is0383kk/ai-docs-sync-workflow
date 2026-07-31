---
read_when:
    - OpenClaw zum ersten Mal einrichten
    - Suche nach gängigen Konfigurationsmustern
    - Zu bestimmten Konfigurationsabschnitten navigieren
summary: 'Konfigurationsübersicht: häufige Aufgaben, Schnelleinrichtung und Links zur vollständigen Referenz'
title: Konfiguration
x-i18n:
    generated_at: "2026-07-26T18:22:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cc04efa16f32e12d6ebcea7a1d36b336df32227fe66953c5d70107708ee6c3
    source_path: gateway/configuration.md
    workflow: 16
---

OpenClaw liest eine optionale <Tooltip tip="JSON5 unterstützt Kommentare und nachgestellte Kommas">**JSON5**</Tooltip>-Konfiguration aus `~/.openclaw/openclaw.json`. Wenn die Datei fehlt, verwendet OpenClaw sichere Standardwerte.

Der aktive Konfigurationspfad muss eine reguläre Datei sein. Von OpenClaw ausgeführte Schreibvorgänge ersetzen sie atomar (durch Umbenennen auf den Pfad), sodass bei einer symbolisch verknüpften `openclaw.json` deren Ziel ersetzt wird, statt durch den Link hindurch zu schreiben – vermeiden Sie Konfigurationslayouts mit symbolischen Links. Wenn Sie die Konfiguration außerhalb des standardmäßigen Zustandsverzeichnisses aufbewahren, lassen Sie `OPENCLAW_CONFIG_PATH` direkt auf die tatsächliche Datei verweisen.

Häufige Gründe für das Hinzufügen einer Konfiguration:

- Kanäle verbinden und festlegen, wer dem Bot Nachrichten senden darf
- Modelle, Tools, Sandboxing oder Automatisierung (Cron, Hooks) festlegen
- Sitzungen, Medien, Netzwerk oder Benutzeroberfläche abstimmen

Alle verfügbaren Felder finden Sie in der [vollständigen Referenz](/de/gateway/configuration-reference).

Für die Konfiguration gilt eine Zwei-Bereiche-Regel: Gleichgeordnete Einträge auf Stammebene enthalten Infrastruktur und agentenübergreifende Standardwerte, während `agents.defaults` das Verhalten der Agentenschleife enthält. Einträge unter `agents.entries` können beide Bereiche überschreiben, sofern das Schema eine agentenspezifische Überschreibung unterstützt.

Agenten und Automatisierungen sollten vor dem Bearbeiten der Konfiguration `config.schema.lookup` für eine genaue Dokumentation auf Feldebene verwenden. Nutzen Sie diese Seite für aufgabenorientierte Anleitungen und die
[Konfigurationsreferenz](/de/gateway/configuration-reference) für die umfassendere
Übersicht der Felder und Standardwerte.

<Tip>
**Neu bei der Konfiguration?** Beginnen Sie mit `openclaw onboard` für die interaktive Einrichtung oder lesen Sie den Leitfaden [Konfigurationsbeispiele](/de/gateway/configuration-examples), der vollständige Konfigurationen zum Kopieren und Einfügen enthält.
</Tip>

## Minimale Konfiguration

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Konfiguration bearbeiten

<Tabs>
  <Tab title="Interaktiver Assistent">
    ```bash
    openclaw onboard       # vollständiger Onboarding-Ablauf
    openclaw configure     # Konfigurationsassistent
    ```
  </Tab>
  <Tab title="CLI (Einzeiler)">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="Steuerungsoberfläche">
    Öffnen Sie [http://127.0.0.1:18789](http://127.0.0.1:18789) und verwenden Sie die Registerkarte **Konfiguration**.
    Die Steuerungsoberfläche erzeugt aus dem aktiven Konfigurationsschema ein Formular, einschließlich der Dokumentationsmetadaten für die Felder
    `title` / `description` sowie der Plugin- und Kanalschemas, sofern
    verfügbar. Als Ausweichmöglichkeit steht ein Editor für **Rohes JSON** bereit. Für Detailansichten
    und andere Tools stellt das Gateway außerdem `config.schema.lookup` bereit, um
    einen einzelnen pfadbezogenen Schemaknoten sowie Zusammenfassungen seiner unmittelbaren untergeordneten Elemente abzurufen.
    In den Einstellungen werden häufig verwendete Felder zuerst angezeigt. Jeder Abschnitt bewahrt seine erweiterten Felder
    in einer eingeklappten Gruppe **Erweitert (N)** auf; verwenden Sie **Erweiterte anzeigen**, um alle
    Gruppen aufzuklappen. Die Einstellungssuche umfasst stets beide Ebenen und öffnet bei Bedarf
    die passende erweiterte Gruppe.
  </Tab>
  <Tab title="Direkte Bearbeitung">
    Bearbeiten Sie `~/.openclaw/openclaw.json` direkt. Das Gateway überwacht die Datei und wendet Änderungen automatisch an (siehe [Hot Reload](#config-hot-reload)).
  </Tab>
</Tabs>

## Strikte Validierung

<Warning>
OpenClaw akzeptiert ausschließlich Konfigurationen, die vollständig dem Schema entsprechen. Unbekannte Schlüssel, fehlerhafte Typen oder ungültige Werte führen dazu, dass das Gateway den **Start verweigert**. Die einzige Ausnahme auf Stammebene ist `$schema` (Zeichenfolge), damit Editoren JSON-Schema-Metadaten anhängen können.
</Warning>

`openclaw config schema` gibt das kanonische JSON-Schema aus, das von der Steuerungsoberfläche
und zur Validierung verwendet wird. `config.schema.lookup` ruft einen einzelnen pfadbezogenen Knoten sowie
Zusammenfassungen untergeordneter Elemente für Tools mit Detailansichten ab. Die Dokumentationsmetadaten der Felder `title`/`description`
werden in verschachtelte Objekte, Platzhalter- (`*`), Array-Element- (`[]`) und `anyOf`/
`oneOf`/`allOf`-Verzweigungen übernommen. Laufzeitschemas für Plugins und Kanäle werden zusammengeführt, sobald die
Manifestregistrierung geladen ist.

Jedes Konfigurationsendfeld besitzt in `uiHints` eine allgemeine oder erweiterte Darstellungsebene.
`advanced: false` kennzeichnet allgemeine Einstellungen und `advanced: true` kennzeichnet erweiterte
Einstellungen. Ein Endfeld erbt die Ebene des nächstgelegenen übergeordneten Elements, wenn es keinen direkten Hinweis besitzt;
Pfade ohne deklariertes übergeordnetes Element sind standardmäßig erweitert. Dies wirkt sich
nur auf die Darstellung aus, nicht auf Validierung, Standardwerte, das Neuladeverhalten oder darauf, ob der Schlüssel festgelegt werden kann.

Wenn die Validierung fehlschlägt:

- Das Gateway startet nicht
- Nur Diagnosebefehle funktionieren (`openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`)
- Führen Sie `openclaw doctor` aus, um die genauen Probleme anzuzeigen
- Führen Sie `openclaw doctor --fix` aus (`--repair` ist dasselbe Flag; `--yes` überspringt Rückfragen), um Reparaturen anzuwenden

Das Gateway bewahrt nach jedem erfolgreichen Start eine vertrauenswürdige Kopie der letzten als funktionierend bekannten Konfiguration auf,
doch weder beim Start noch beim Hot Reload wird sie automatisch wiederhergestellt – dies geschieht nur durch `openclaw doctor --fix`.
Wenn die Validierung von `openclaw.json` fehlschlägt (einschließlich der Plugin-internen Validierung), schlägt der Start des Gateways
fehl oder das Neuladen wird übersprungen, und die aktuelle Laufzeit verwendet weiterhin die zuletzt akzeptierte
Konfiguration. Ein abgelehnter Schreibvorgang wird zur Überprüfung außerdem als `<path>.rejected.<timestamp>` gespeichert.
Das Gateway blockiert Schreibvorgänge, die wie ein versehentliches Überschreiben aussehen – das Entfernen von `gateway.mode`,
der Verlust des Blocks `meta` oder eine Verkleinerung der Datei um mehr als die Hälfte –, sofern der Schreibvorgang
destruktive Änderungen nicht ausdrücklich zulässt. Die Übernahme als letzte als funktionierend bekannte Konfiguration wird übersprungen, wenn ein
Kandidat einen Platzhalter für ein geschwärztes Geheimnis wie `***` oder `[redacted]` enthält.

## Häufige Aufgaben

<AccordionGroup>
  <Accordion title="Einen Kanal einrichten (WhatsApp, Telegram, Discord usw.)">
    Jeder Kanal besitzt unter `channels.<provider>` einen eigenen Konfigurationsabschnitt. Die Schritte zur Einrichtung finden Sie auf der jeweiligen Kanalseite:

    - [Discord](/de/channels/discord) – `channels.discord`
    - [Feishu](/de/channels/feishu) – `channels.feishu`
    - [Google Chat](/de/channels/googlechat) – `channels.googlechat`
    - [iMessage](/de/channels/imessage) – `channels.imessage`
    - [Mattermost](/de/channels/mattermost) – `channels.mattermost`
    - [Microsoft Teams](/de/channels/msteams) – `channels.msteams`
    - [Signal](/de/channels/signal) – `channels.signal`
    - [Slack](/de/channels/slack) – `channels.slack`
    - [Telegram](/de/channels/telegram) – `channels.telegram`
    - [WhatsApp](/de/channels/whatsapp) – `channels.whatsapp`

    Alle Kanäle verwenden dasselbe Muster für DM-Richtlinien:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // pairing | allowlist | open | disabled
          allowFrom: ["tg:123"], // nur für allowlist/open
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Modelle auswählen und konfigurieren">
    Legen Sie das primäre Modell und optionale Ausweichmodelle fest:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models` speichert Aliasse und modellspezifische Einstellungen; das Hinzufügen eines Eintrags schränkt Überschreibungen durch `/model` oder `--model` niemals ein.
    - `agents.defaults.modelPolicy.allow` ist die explizite Zulassungsliste für Überschreibungen und Modellauswahlfelder. Sie akzeptiert exakte Referenzen und `provider/*`-Platzhalter; lassen Sie sie weg oder verwenden Sie `[]`, um jedes Modell zuzulassen.
    - Modellreferenzen verwenden das Format `provider/model` (z. B. `anthropic/claude-opus-4-6`).
    - `agents.defaults.imageMaxDimensionPx` steuert die Herunterskalierung von Bildern in Transkripten und Tools (Standardwert `1200`); niedrigere Werte reduzieren bei Durchläufen mit vielen Screenshots üblicherweise die Nutzung von Vision-Tokens.
    - Informationen zum Wechseln von Modellen im Chat finden Sie unter [Modelle-CLI](/de/concepts/models), Informationen zur Rotation der Authentifizierung und zum Ausweichverhalten unter [Modell-Failover](/de/concepts/model-failover).
    - Informationen zu benutzerdefinierten oder selbst gehosteten Providern finden Sie unter [Benutzerdefinierte Provider](/de/gateway/config-tools#custom-providers-and-base-urls) in der Referenz.

  </Accordion>

  <Accordion title="Festlegen, wer dem Bot Nachrichten senden darf">
    Der DM-Zugriff wird pro Kanal über `dmPolicy` gesteuert (Standardwert `"pairing"`):

    - `"pairing"`: Unbekannte Absender erhalten einen einmaligen Kopplungscode zur Genehmigung
    - `"allowlist"`: Nur Absender in `allowFrom` (oder im Speicher für gekoppelte Zulassungen)
    - `"open"`: Alle eingehenden DMs zulassen (erfordert `allowFrom: ["*"]`)
    - `"disabled"`: Alle DMs ignorieren

    Verwenden Sie für Gruppen `groupPolicy` (`"allowlist" | "open" | "disabled"`) zusammen mit `groupAllowFrom` oder kanalspezifischen Zulassungslisten.

    Kanalspezifische Details finden Sie in der [vollständigen Referenz](/de/gateway/config-channels#dm-and-group-access).

  </Accordion>

  <Accordion title="Erwähnungssperre für Gruppenchats einrichten">
    Für Gruppennachrichten ist standardmäßig eine **Erwähnung erforderlich**. Konfigurieren Sie die Auslösemuster pro Agent. Normale Gruppen- und Kanalantworten werden automatisch veröffentlicht; aktivieren Sie für gemeinsam genutzte Räume, in denen der Agent selbst entscheiden soll, wann er spricht, ausdrücklich den Pfad über das Nachrichten-Tool:

    ```json5
    {
      messages: {
        visibleReplies: "automatic", // auf "message_tool" setzen, um überall Sends über das Nachrichten-Tool zu verlangen
        groupChat: {
          visibleReplies: "message_tool", // Opt-in; sichtbare Ausgabe erfordert message(action=send)
          unmentionedInbound: "room_event", // nicht erwähnte, ständig aktive Gruppenunterhaltung dient als stiller Kontext
        },
      },
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **Metadaten-Erwähnungen**: native @-Erwähnungen (Antippen zum Erwähnen in WhatsApp, Telegram @bot usw.)
    - **Textmuster**: sichere reguläre Ausdrücke in `mentionPatterns`
    - **Sichtbare Antworten**: `messages.visibleReplies` kann Sends über das Nachrichten-Tool global verlangen; `messages.groupChat.visibleReplies` überschreibt dies für Gruppen/Kanäle.
    - Informationen zu Modi für sichtbare Antworten, kanalspezifischen Überschreibungen und dem Selbstchatmodus finden Sie in der [vollständigen Referenz](/de/gateway/config-channels#group-chat-mention-gating).

  </Accordion>

  <Accordion title="Skills pro Agent einschränken">
    Verwenden Sie `agents.defaults.skills` für eine gemeinsame Ausgangsbasis und überschreiben Sie anschließend bestimmte
    Agenten mit `agents.entries.*.skills`:

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // erbt github, weather
          { id: "docs", skills: ["docs-search"] }, // ersetzt Standardwerte
          { id: "locked-down", skills: [] }, // keine Skills
        ],
      },
    }
    ```

    - Lassen Sie `agents.defaults.skills` weg, um Skills standardmäßig nicht einzuschränken.
    - Lassen Sie `agents.entries.*.skills` weg, um die Standardwerte zu erben.
    - Legen Sie `agents.entries.*.skills: []` fest, um keine Skills zuzulassen.
    - Weitere Informationen finden Sie unter [Skills](/de/tools/skills), [Skills-Konfiguration](/de/tools/skills-config) und
      in der [Konfigurationsreferenz](/de/gateway/config-agents#agents-defaults-skills).

  </Accordion>

  <Accordion title="Kanalspezifische Zustandsüberwachung konfigurieren">
    Deaktivieren oder aktivieren Sie automatische Neustarts bei Zustandsproblemen für einen Kanal oder ein Konto:

    ```json5
    {
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - Verwenden Sie `channels.<provider>.healthMonitor.enabled` oder `channels.<provider>.accounts.<id>.healthMonitor.enabled`, um automatische Neustarts für einen einzelnen Kanal oder ein einzelnes Konto zu steuern.
    - Informationen zur betrieblichen Fehlerdiagnose finden Sie unter [Zustandsprüfungen](/de/gateway/health), alle Felder in der [vollständigen Referenz](/de/gateway/configuration-reference#gateway).

  </Accordion>

  <Accordion title="Sitzungen und Zurücksetzungen konfigurieren">
    Sitzungen steuern die Kontinuität und Isolation von Unterhaltungen:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // für mehrere Benutzer empfohlen
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main` (gemeinsam genutzt) | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: globale Standardwerte für das Routing Thread-gebundener Sitzungen. `/focus`, `/unfocus`, `/agents`, `/session idle` und `/session max-age` binden, lösen, listen und konfigurieren dies pro Sitzung (Discord bindet Threads, Telegram bindet Themen/Unterhaltungen).
    - Informationen zu Geltungsbereichen, Identitätsverknüpfungen und Senderichtlinien finden Sie unter [Sitzungsverwaltung](/de/concepts/session).
    - Alle Felder finden Sie in der [vollständigen Referenz](/de/gateway/config-agents#session).

  </Accordion>

  <Accordion title="Sandboxing aktivieren">
    Führen Sie Agentensitzungen in isolierten Sandbox-Laufzeitumgebungen aus:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    Erstellen Sie zuerst das Image: Führen Sie aus einem Quellcode-Checkout `scripts/sandbox-setup.sh` aus oder verwenden Sie bei einer npm-Installation den eingebetteten Befehl `docker build` unter [Sandboxing § Images und Einrichtung](/de/gateway/sandboxing#images-and-setup).

    Die vollständige Anleitung finden Sie unter [Sandboxing](/de/gateway/sandboxing), alle Optionen in der [vollständigen Referenz](/de/gateway/config-agents#agentsdefaultssandbox).

  </Accordion>

  <Accordion title="Relay-gestützte Push-Benachrichtigungen für offizielle iOS-Builds aktivieren">
    Relay-gestützte Push-Benachrichtigungen für öffentliche App-Store-Builds verwenden das gehostete OpenClaw-Relay: `https://ios-push-relay.openclaw.ai`.

    Benutzerdefinierte Relay-Bereitstellungen erfordern einen bewusst getrennten iOS-Build- und Bereitstellungspfad, dessen Relay-URL mit der Relay-URL des Gateways übereinstimmt. Wenn Sie einen benutzerdefinierten Relay-Build verwenden, legen Sie Folgendes in der Gateway-Konfiguration fest:

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // Optional. Standardwert: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    Entsprechender CLI-Befehl:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    Funktionsweise:

    - Ermöglicht dem Gateway, `push.test`, Aktivierungsimpulse und Aktivierungen zur Wiederverbindung über das externe Relay zu senden.
    - Verwendet eine auf die Registrierung beschränkte Sendeberechtigung, die von der gekoppelten iOS-App weitergeleitet wird. Das Gateway benötigt kein bereitstellungsweites Relay-Token.
    - Bindet jede Relay-gestützte Registrierung an die Gateway-Identität, mit der die iOS-App gekoppelt wurde, sodass kein anderes Gateway die gespeicherte Registrierung wiederverwenden kann.
    - Lokale/manuelle iOS-Builds verwenden weiterhin direkte APNs. Relay-gestütztes Senden gilt nur für offiziell verteilte Builds, die über das Relay registriert wurden.
    - Muss mit der im iOS-Build eingebetteten Relay-Basis-URL übereinstimmen, damit Registrierungs- und Sendedatenverkehr dieselbe Relay-Bereitstellung erreichen.

    End-to-End-Ablauf:

    1. Installieren Sie die offizielle iOS-App.
    2. Optional: Konfigurieren Sie `gateway.push.apns.relay.baseUrl` auf dem Gateway nur, wenn Sie einen bewusst getrennten benutzerdefinierten Relay-Build verwenden.
    3. Koppeln Sie die iOS-App mit dem Gateway und lassen Sie sowohl Node- als auch Operatorsitzungen eine Verbindung herstellen.
    4. Die iOS-App ruft die Gateway-Identität ab, registriert sich mittels App Attest und App-Beleg beim Relay und veröffentlicht anschließend die Relay-gestützte `push.apns.register`-Nutzlast an das gekoppelte Gateway.
    5. Das Gateway speichert das Relay-Handle und die Sendeberechtigung und verwendet sie anschließend für `push.test`, Aktivierungsimpulse und Aktivierungen zur Wiederverbindung.

    Betriebshinweise:

    - Wenn Sie die iOS-App auf ein anderes Gateway umstellen, verbinden Sie die App erneut, damit sie eine neue, an dieses Gateway gebundene Relay-Registrierung veröffentlichen kann.
    - Wenn Sie einen neuen iOS-Build ausliefern, der auf eine andere Relay-Bereitstellung verweist, aktualisiert die App ihre zwischengespeicherte Relay-Registrierung, anstatt den alten Relay-Ursprung wiederzuverwenden.

    Kompatibilitätshinweis:

    - `OPENCLAW_APNS_RELAY_BASE_URL` und `OPENCLAW_APNS_RELAY_TIMEOUT_MS` funktionieren weiterhin als temporäre Umgebungsüberschreibungen.
    - Benutzerdefinierte Gateway-Relay-URLs müssen mit der im iOS-Build eingebetteten Relay-Basis-URL übereinstimmen; der öffentliche App-Store-Release-Kanal lehnt benutzerdefinierte Überschreibungen der iOS-Relay-URL ab.
    - `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` bleibt ein ausschließlich für Loopback bestimmter Entwicklungsnotausgang; speichern Sie HTTP-Relay-URLs nicht dauerhaft in der Konfiguration.

    Den End-to-End-Ablauf finden Sie unter [iOS-App](/de/platforms/ios#relay-backed-push-for-official-builds), das Relay-Sicherheitsmodell unter [Authentifizierungs- und Vertrauensablauf](/de/platforms/ios#authentication-and-trust-flow).

  </Accordion>

  <Accordion title="Heartbeat einrichten (regelmäßige Statusmeldungen)">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: Zeitdauerzeichenfolge (`30m`, `2h`). Setzen Sie `0m`, um die Funktion zu deaktivieren. Standardwert: `30m`.
    - `target`: `last` | `none` | `<channel-id>` (zum Beispiel `discord`, `matrix`, `telegram` oder `whatsapp`)
    - `directPolicy`: `allow` (Standardwert) oder `block` für DM-artige Heartbeat-Ziele
    - Die vollständige Anleitung finden Sie unter [Heartbeat](/de/gateway/heartbeat).

  </Accordion>

  <Accordion title="Cron-Aufgaben konfigurieren">
    ```json5
    {
      cron: {
        enabled: true,
        sessionRetention: "24h",
      },
    }
    ```

    - `sessionRetention`: Entfernt abgeschlossene isolierte Ausführungssitzungen aus SQLite-Sitzungszeilen (Standardwert `24h`; zum Deaktivieren `false` festlegen).
    - Der Ausführungsverlauf bewahrt automatisch die neuesten 2000 Terminalzeilen pro Aufgabe auf; verlorene Zeilen behalten ihr 24-stündiges Bereinigungsfenster.
    - Eine Funktionsübersicht und CLI-Beispiele finden Sie unter [Cron-Aufgaben](/de/automation/cron-jobs).

  </Accordion>

  <Accordion title="Webhooks (Hooks) einrichten">
    Aktivieren Sie HTTP-Webhook-Endpunkte auf dem Gateway:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    Sicherheitshinweis:
    - Behandeln Sie sämtliche Hook-/Webhook-Nutzlastinhalte als nicht vertrauenswürdige Eingaben.
    - Verwenden Sie ein dediziertes `hooks.token`; verwenden Sie keine aktiven Gateway-Authentifizierungsgeheimnisse erneut (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` oder `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`).
    - Die Hook-Authentifizierung erfolgt ausschließlich über Header (`Authorization: Bearer ...` oder `x-openclaw-token`); Token in Abfragezeichenfolgen werden abgelehnt.
    - `hooks.path` darf nicht `/` sein; belassen Sie den Webhook-Eingang auf einem dedizierten Unterpfad wie `/hooks`.
    - Lassen Sie Flags zum Umgehen der Prüfung unsicherer Inhalte deaktiviert (`hooks.gmail.allowUnsafeExternalContent`, `hooks.mappings[].allowUnsafeExternalContent`), außer bei eng begrenzter Fehlerdiagnose.
    - Wenn Sie `hooks.allowRequestSessionKey` aktivieren, legen Sie außerdem `hooks.allowedSessionKeyPrefixes` fest, um die vom Aufrufer gewählten Sitzungsschlüssel einzugrenzen.
    - Bevorzugen Sie für Hook-gesteuerte Agenten leistungsfähige moderne Modellstufen und strenge Tool-Richtlinien (zum Beispiel ausschließlich Messaging sowie nach Möglichkeit Sandboxing).

    Alle Zuordnungsoptionen und die Gmail-Integration finden Sie in der [vollständigen Referenz](/de/gateway/configuration-reference#hooks).

  </Accordion>

  <Accordion title="Multi-Agent-Routing konfigurieren">
    Führen Sie mehrere isolierte Agenten mit getrennten Arbeitsbereichen und Sitzungen aus:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    Bindungsregeln und agentenspezifische Zugriffsprofile finden Sie unter [Multi-Agent](/de/concepts/multi-agent) und in der [vollständigen Referenz](/de/gateway/config-agents#multi-agent-routing).

  </Accordion>

  <Accordion title="Konfiguration auf mehrere Dateien aufteilen ($include)">
    Verwenden Sie `$include`, um umfangreiche Konfigurationen zu organisieren:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **Einzelne Datei**: Ersetzt das enthaltende Objekt
    - **Datei-Array**: Wird der Reihe nach tief zusammengeführt (spätere Werte haben Vorrang), bis zu 10 verschachtelte Ebenen
    - **Schwesterschlüssel**: Werden nach den Einbindungen zusammengeführt (überschreiben eingebundene Werte)
    - **Relative Pfade**: Werden relativ zur einbindenden Datei aufgelöst
    - **Pfadformat**: Einbindungspfade dürfen keine Nullbytes enthalten und müssen vor und nach der Auflösung strikt kürzer als 4096 Zeichen sein
    - **OpenClaw-eigene Schreibvorgänge**: Wenn ein Schreibvorgang nur einen Abschnitt der obersten Ebene ändert,
      der durch eine Einbindung einer einzelnen Datei wie `plugins: { $include: "./plugins.json5" }` gestützt wird,
      aktualisiert OpenClaw diese eingebundene Datei und lässt `openclaw.json` unverändert
    - **Nicht unterstütztes Durchschreiben**: Root-Einbindungen, Einbindungs-Arrays und Einbindungen
      mit Schwesterschlüssel-Überschreibungen schlagen bei OpenClaw-eigenen Schreibvorgängen sicher fehl,
      anstatt die Konfiguration zu verflachen
    - **Einschränkung**: `$include`-Pfade müssen unterhalb des Verzeichnisses aufgelöst werden, das
      `openclaw.json` enthält. Um einen Verzeichnisbaum über Computer oder Benutzer hinweg gemeinsam zu verwenden, legen Sie
      `OPENCLAW_INCLUDE_ROOTS` auf eine Pfadliste (`:` unter POSIX, `;` unter Windows) mit
      zusätzlichen Verzeichnissen fest, auf die Einbindungen verweisen dürfen. Symbolische Links werden aufgelöst
      und erneut geprüft. Daher wird ein Pfad weiterhin abgelehnt, der lexikalisch in einem Konfigurationsverzeichnis liegt,
      dessen tatsächliches Ziel jedoch aus allen zulässigen Wurzelverzeichnissen herausführt.
    - **Fehlerbehandlung**: Eindeutige Fehler bei fehlenden Dateien, Analysefehlern, zirkulären Einbindungen, ungültigem Pfadformat und übermäßiger Länge

  </Accordion>
</AccordionGroup>

## Automatisches Neuladen der Konfiguration

Das Gateway überwacht `~/.openclaw/openclaw.json` und wendet Änderungen automatisch an – für die meisten Einstellungen ist kein manueller Neustart erforderlich.

Direkte Dateiänderungen gelten bis zur erfolgreichen Validierung als nicht vertrauenswürdig. Die Überwachung wartet,
bis temporäre Schreib- und Umbenennungsvorgänge des Editors abgeschlossen sind, liest die endgültige Datei und lehnt
ungültige externe Änderungen ab, ohne `openclaw.json` neu zu schreiben. OpenClaw-eigene Konfigurationsschreibvorgänge
verwenden vor dem Schreiben dieselbe Schemaprüfung (die für jeden Schreibvorgang geltenden Regeln zum Überschreiben und
Zurücksetzen finden Sie unter [Strikte Validierung](#strict-validation)).

Wenn `config reload skipped (invalid config)` angezeigt wird oder beim Start `Invalid
config` gemeldet wird, prüfen Sie die Konfiguration, führen Sie `openclaw config validate` und anschließend zur Reparatur `openclaw
doctor --fix` aus. Die Prüfliste finden Sie unter [Gateway-Fehlerbehebung](/de/gateway/troubleshooting#gateway-rejected-invalid-config).

### Neulademodi

| Modus                   | Verhalten                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **`hybrid`** (Standard) | Wendet sichere Änderungen sofort im laufenden Betrieb an. Bei kritischen Änderungen erfolgt automatisch ein Neustart.           |
| **`hot`**              | Wendet nur sichere Änderungen im laufenden Betrieb an. Protokolliert eine Warnung, wenn ein Neustart erforderlich ist – Sie führen ihn durch. |
| **`restart`**          | Startet das Gateway bei jeder Konfigurationsänderung neu, unabhängig davon, ob sie sicher ist.                                 |
| **`off`**              | Deaktiviert die Dateiüberwachung. Änderungen werden beim nächsten manuellen Neustart wirksam.                 |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### Was im laufenden Betrieb angewendet wird und was einen Neustart erfordert

Die meisten Felder werden ohne Ausfallzeit im laufenden Betrieb angewendet; bei einigen entsprechenden Abschnitten wird nur das jeweilige
Subsystem (Kanal, Cron, Heartbeat, Zustandsmonitor) statt des gesamten Gateways neu gestartet. Im
Modus `hybrid` werden Änderungen, die einen Neustart des Gateways erfordern, automatisch verarbeitet.

| Kategorie            | Felder                                                                  | Gateway-Neustart erforderlich?      |
| ------------------- | ----------------------------------------------------------------------- | ---------------------------- |
| Kanäle            | `channels.*`, `web` (WhatsApp) – alle integrierten und Plugin-Kanäle       | Nein (startet diesen Kanal neu)   |
| Agent und Modelle      | `agent`, `agents`, `models`, `routing`                                  | Nein                           |
| Automatisierung          | `hooks`, `cron`, `agent.heartbeat`                                      | Nein (startet dieses Subsystem neu) |
| Sitzungen und Nachrichten | `session`, `messages`                                                   | Nein                           |
| Tools und Medien       | `tools`, `skills`, `mcp`, `audio`, `talk`                               | Nein                           |
| Plugin-Konfiguration       | `plugins.entries.*`, `plugins.allow`, `plugins.deny`, `plugins.enabled` | Nein (lädt die Plugin-Laufzeit neu)  |
| Benutzeroberfläche und Sonstiges           | `ui`, `logging`, `identity`, `bindings`                                 | Nein                           |
| Gateway-Server      | `gateway.*` (Port, Bindung, Authentifizierung, Tailscale, TLS, HTTP, Push)              | **Ja**                      |
| Infrastruktur      | `discovery`, `browser`, `plugins.load`, `plugins.installs`              | **Ja**                      |

<Note>
`gateway.reload` und `gateway.remote` sind unter `gateway.*` Ausnahmen – ihre Änderung löst **keinen** Neustart aus. Einzelne Plugins können diese Tabelle ebenfalls überschreiben: Ein geladenes Plugin kann eigene Konfigurationspräfixe deklarieren, die einen Neustart auslösen (beispielsweise startet das gebündelte Canvas-Plugin das Gateway für `plugins.enabled`, `plugins.allow` und `plugins.deny` neu, nicht nur für sein eigenes `plugins.entries.canvas`). Das tatsächliche Verhalten hängt daher davon ab, welche Plugins aktiv sind.
</Note>

### Planung des Neuladens

Wenn Sie eine Quelldatei bearbeiten, auf die über `$include` verwiesen wird, plant OpenClaw
das Neuladen anhand der in den Quellen definierten Struktur und nicht anhand der abgeflachten Ansicht im Arbeitsspeicher.
Dadurch bleiben Entscheidungen zum Neuladen im laufenden Betrieb (Anwendung im laufenden Betrieb oder Neustart) vorhersehbar, selbst wenn sich ein
einzelner Abschnitt der obersten Ebene in einer eigenen eingebundenen Datei befindet, beispielsweise
`plugins: { $include: "./plugins.json5" }`. Die Planung des Neuladens schlägt sicher geschlossen fehl, wenn die
Quellstruktur mehrdeutig ist.

## Konfigurations-RPC (programmatische Aktualisierungen)

Für Tools, die Konfigurationen über die Gateway-API schreiben, wird dieser Ablauf empfohlen:

- `config.schema.lookup`, um einen Teilbaum zu untersuchen (flacher Schemaknoten und Zusammenfassungen
  der untergeordneten Elemente)
- `config.get`, um den aktuellen Snapshot zusammen mit `hash` abzurufen
- `config.patch` für partielle Aktualisierungen (JSON-Merge-Patch: Objekte werden zusammengeführt, `null`
  löscht, Arrays werden ersetzt, wenn dies mit `replacePaths` ausdrücklich bestätigt wird, falls
  Einträge entfernt würden)
- `config.apply` nur, wenn die gesamte Konfiguration ersetzt werden soll
- `update.run` für eine ausdrückliche Selbstaktualisierung mit anschließendem Neustart; fügen Sie `continuationMessage` hinzu, wenn die Sitzung nach dem Neustart einen weiteren Durchlauf ausführen soll
- `update.status`, um den neuesten Neustart-Sentinel der Aktualisierung zu untersuchen und nach einem Neustart die ausgeführte Version zu überprüfen

Agenten sollten `config.schema.lookup` als erste Anlaufstelle für genaue
Dokumentation und Einschränkungen auf Feldebene verwenden. Verwenden Sie die [Konfigurationsreferenz](/de/gateway/configuration-reference),
wenn eine umfassendere Konfigurationsübersicht, Standardwerte oder Links zu speziellen
Subsystemreferenzen benötigt werden.

<Note>
Schreibvorgänge der Steuerungsebene (`config.apply`, `config.patch`, `update.run`) sind
pro Methode und pro `deviceId+clientIp` auf 30 Anfragen je 60 Sekunden
begrenzt; siehe [Ratenbegrenzung](/de/gateway/security/rate-limiting). Neustartanforderungen
werden zusammengeführt; anschließend gilt zwischen Neustartzyklen eine Abkühlzeit von 30 Sekunden.
`update.status` ist schreibgeschützt, jedoch auf Administratoren beschränkt, da der Neustart-Sentinel
Zusammenfassungen der Aktualisierungsschritte und die letzten Zeilen der Befehlsausgabe enthalten kann.
</Note>

Beispiel für einen partiellen Patch:

```bash
openclaw gateway call config.get --params '{}'  # payload.hash erfassen
openclaw gateway call config.patch --params '{
  "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
  "baseHash": "<hash>"
}'
```

Sowohl `config.apply` als auch `config.patch` akzeptieren `raw`, `baseHash`, `sessionKey`,
`note` und `restartDelayMs`. `baseHash` ist für beide Methoden erforderlich, sobald bereits eine
Konfigurationsdatei vorhanden ist (beim ersten Schreibvorgang ohne vorhandene Konfiguration wird die Prüfung übersprungen).

`config.patch` akzeptiert außerdem `replacePaths`, ein Array von Konfigurationspfaden, deren Array-Ersetzung
beabsichtigt ist. Wenn ein Patch ein vorhandenes Array durch eines mit weniger Einträgen ersetzen oder es
löschen würde, lehnt das Gateway den Schreibvorgang ab, sofern dieser genaue Pfad nicht in
`replacePaths` enthalten ist; verschachtelte Arrays innerhalb von Array-Einträgen verwenden `[]`, beispielsweise
`agents.entries.*.skills`. Dadurch wird verhindert, dass verkürzte `config.get`-Snapshots
unbemerkt Routing- oder Zulassungslisten-Arrays überschreiben. Verwenden Sie `config.apply`, wenn die
gesamte Konfiguration ersetzt werden soll.

## Umgebungsvariablen

OpenClaw liest Umgebungsvariablen aus dem übergeordneten Prozess sowie aus:

- `.env` aus dem aktuellen Arbeitsverzeichnis (falls vorhanden)
- `~/.openclaw/.env` (globaler Rückgriff)

Keine der beiden Dateien überschreibt vorhandene Umgebungsvariablen. Sie können Umgebungsvariablen auch direkt in der Konfiguration festlegen:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="Import der Shell-Umgebung (optional)">
  Wenn diese Option aktiviert ist und erwartete Schlüssel nicht gesetzt sind, führt OpenClaw Ihre Anmelde-Shell aus und importiert nur die fehlenden Schlüssel:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

Entsprechende Umgebungsvariable: `OPENCLAW_LOAD_SHELL_ENV=1`. Standardwert für `timeoutMs`: `15000`.
</Accordion>

<Accordion title="Ersetzung von Umgebungsvariablen in Konfigurationswerten">
  Verweisen Sie in einem beliebigen Zeichenfolgenwert der Konfiguration mit `${VAR_NAME}` auf Umgebungsvariablen:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

Regeln:

- Es werden nur Namen in Großbuchstaben berücksichtigt: `[A-Z_][A-Z0-9_]*`
- Fehlende oder leere Variablen lösen beim Laden einen Fehler aus
- Für eine literale Ausgabe mit `$${VAR}` maskieren
- Funktioniert innerhalb von `$include`-Dateien
- Inline-Ersetzung: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="Geheimnisreferenzen (Umgebung, Datei, Ausführung)">
  Für Felder, die SecretRef-Objekte unterstützen, können Sie Folgendes verwenden:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccount: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

Details zu SecretRef (einschließlich `secrets.providers` für `env`/`file`/`exec`) finden Sie unter [Verwaltung von Geheimnissen](/de/gateway/secrets).
Unterstützte Anmeldedatenpfade sind unter [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface) aufgeführt.
</Accordion>

Die vollständige Rangfolge und alle Quellen finden Sie unter [Umgebung](/de/help/environment).

## Vollständige Referenz

Die vollständige Referenz für jedes einzelne Feld finden Sie in der **[Konfigurationsreferenz](/de/gateway/configuration-reference)**.

---

_Zugehörige Themen: [Konfigurationsbeispiele](/de/gateway/configuration-examples) · [Konfigurationsreferenz](/de/gateway/configuration-reference) · [Doctor](/de/gateway/doctor)_

## Verwandte Themen

- [Konfigurationsreferenz](/de/gateway/configuration-reference)
- [Konfigurationsbeispiele](/de/gateway/configuration-examples)
- [Gateway-Betriebshandbuch](/de/gateway)
