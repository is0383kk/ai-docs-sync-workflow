---
read_when:
    - Arbeiten am Gateway-Protokoll, an Clients oder Übertragungswegen
summary: WebSocket-Gateway-Architektur, Komponenten und Client-Abläufe
title: Gateway-Architektur
x-i18n:
    generated_at: "2026-07-26T17:46:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8054bd87f738b957c24f8d6965d55365de2293d44902530a9ba778afa597cc7
    source_path: concepts/architecture.md
    workflow: 16
---

## Übersicht

- Ein einzelner langlebiger **Gateway** verwaltet alle Messaging-Oberflächen (WhatsApp über
  Baileys, Telegram über grammY, Slack, Discord, Signal, iMessage, WebChat).
- Steuerungsebenen-Clients (macOS-App, CLI, Web-UI, Automatisierungen) verbinden sich über
  **WebSocket** auf dem konfigurierten Bind-Host mit dem Gateway (Standard:
  `127.0.0.1:18789`).
- **Nodes** (macOS/iOS/Android/headless) verbinden sich ebenfalls über **WebSocket**, deklarieren jedoch
  `role: node` mit expliziten Funktionen/Befehlen.
- Ein Gateway pro Host; nur dieser öffnet eine WhatsApp-Sitzung.
- Der **Canvas-Host** wird vom Gateway-HTTP-Server unter folgenden Pfaden bereitgestellt:
  - `/__openclaw__/canvas/` (vom Agenten bearbeitbares HTML/CSS/JS)
  - `/__openclaw__/a2ui/` (A2UI-Host)

  Er verwendet denselben Port wie der Gateway (Standard: `18789`).

## Komponenten und Abläufe

### Gateway (Daemon)

- Verwaltet Provider-Verbindungen.
- Stellt eine typisierte WS-API bereit (Anfragen, Antworten, Server-Push-Ereignisse).
- Validiert eingehende Frames anhand eines JSON-Schemas.
- Gibt Ereignisse wie `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron` aus.

### Clients (Mac-App / CLI / Webadministration)

- Eine WS-Verbindung pro Client.
- Senden Anfragen (`health`, `status`, `send`, `agent`, `system-presence`).
- Abonnieren Ereignisse (`tick`, `agent`, `presence`, `shutdown`).

### Nodes (macOS / iOS / Android / headless)

- Verbinden sich mit `role: node` mit **demselben WS-Server**.
- Stellen in `connect` eine Geräteidentität bereit; das Pairing ist **gerätebasiert** (Rolle `node`) und
  die Genehmigung wird im Geräte-Pairing-Speicher verwaltet.
- Stellen Befehle wie `canvas.*`, `camera.*`, `screen.record`, `location.get` bereit.

Protokolldetails: [Gateway-Protokoll](/de/gateway/protocol)

### WebChat

- Statische UI, die die Gateway-WS-API für den Chatverlauf und zum Senden verwendet.
- Stellt bei Remote-Einrichtungen die Verbindung über denselben SSH-/Tailscale-Tunnel wie andere
  Clients her.

## Verbindungslebenszyklus (einzelner Client)

```mermaid
sequenceDiagram
    participant Client
    participant Gateway

    Client->>Gateway: Anfr.:connect
    Gateway-->>Client: Antw. (ok)
    Note right of Gateway: oder Antwortfehler + Schließen
    Note left of Client: payload=hello-ok<br>Momentaufnahme: Präsenz + Zustand

    Gateway-->>Client: Ereignis:presence
    Gateway-->>Client: Ereignis:tick

    Client->>Gateway: Anfr.:agent
    Gateway-->>Client: Antw.:agent<br>Bestätigung {runId, status:"accepted"}
    Gateway-->>Client: Ereignis:agent<br>(Streaming)
    Gateway-->>Client: Antw.:agent<br>abschließend {runId, status, summary}
```

## Übertragungsprotokoll (Zusammenfassung)

- Transport: WebSocket, Text-Frames mit JSON-Nutzdaten.
- Der erste Frame **muss** `connect` sein.
- Nach dem Handshake:
  - Anfragen: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - Ereignisse: `{type:"event", event, payload, seq?, stateVersion?}`
- `hello-ok.features.methods` / `events` sind Metadaten zur Erkennung und kein
  generierter Auszug aller aufrufbaren Hilfsrouten.
- Die Authentifizierung mit einem gemeinsamen Geheimnis verwendet je nach konfiguriertem
  Gateway-Authentifizierungsmodus `connect.params.auth.token` oder `connect.params.auth.password`.
- Identitätsführende Modi wie Tailscale Serve
  (`gateway.auth.allowTailscale: true`) oder `gateway.auth.mode: "trusted-proxy"` außerhalb des Loopbacks
  erfüllen die Authentifizierung anhand der Anfrage-Header
  anstelle von `connect.params.auth.*`.
- `gateway.auth.mode: "none"` für privaten Ingress deaktiviert die Authentifizierung mit gemeinsamem Geheimnis
  vollständig; lassen Sie diesen Modus bei öffentlichem/nicht vertrauenswürdigem Ingress deaktiviert.
- Idempotenzschlüssel sind für Methoden mit Nebenwirkungen (`send`, `agent`) erforderlich, damit
  Wiederholungsversuche sicher sind; der Server hält einen kurzlebigen Deduplizierungs-Cache vor.
- Nodes müssen `role: "node"` sowie Funktionen/Befehle/Berechtigungen in `connect` enthalten.

## Pairing und lokales Vertrauen

- Alle WS-Clients (Operatoren + Nodes) enthalten bei `connect` eine **Geräteidentität**.
- Neue Geräte-IDs erfordern eine Pairing-Genehmigung; der Gateway stellt ein **Geräte-Token**
  für nachfolgende Verbindungen aus.
- Direkte lokale Loopback-Verbindungen können automatisch genehmigt werden, um auf demselben Host
  eine reibungslose Benutzererfahrung zu gewährleisten.
- OpenClaw verfügt außerdem über einen eng begrenzten, Backend-/Container-lokalen Selbstverbindungspfad für
  vertrauenswürdige Hilfsabläufe mit gemeinsamem Geheimnis.
- Tailnet- und LAN-Verbindungen, einschließlich Tailnet-Bindungen auf demselben Host, erfordern weiterhin
  eine explizite Pairing-Genehmigung.
- Alle Verbindungen müssen die Nonce `connect.challenge` signieren. Die Signaturnutzdaten `v3`
  binden außerdem `platform` und `deviceFamily`; der Gateway fixiert beim erneuten
  Verbinden die gekoppelten Metadaten und verlangt bei Metadatenänderungen ein erneutes Pairing.
- **Nicht lokale** Verbindungen erfordern weiterhin eine explizite Genehmigung.
- Die Gateway-Authentifizierung (`gateway.auth.*`) gilt weiterhin für **alle** Verbindungen, unabhängig davon, ob sie lokal oder
  remote sind.

Details: [Gateway-Protokoll](/de/gateway/protocol), [Pairing](/de/channels/pairing),
[Sicherheit](/de/gateway/security).

## Protokolltypisierung und Codegenerierung

- TypeBox-Schemas definieren das Protokoll.
- Aus diesen Schemas wird ein JSON-Schema generiert.
- Aus dem JSON-Schema werden Swift-Modelle generiert.

## Remote-Zugriff

- Bevorzugt: Tailscale oder VPN.
- Alternative: SSH-Tunnel

  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
  ```

- Über den Tunnel gelten derselbe Handshake und dasselbe Authentifizierungs-Token.
- TLS und optionales Pinning können bei Remote-Einrichtungen für WS aktiviert werden.

## Betriebsübersicht

- Start: `openclaw gateway` (im Vordergrund, Protokollierung nach stdout).
- Zustand: `health` über WS (auch in `hello-ok` enthalten).
- Überwachung: launchd/systemd für automatische Neustarts.

## Invarianten

- Genau ein Gateway steuert pro Host eine einzelne Baileys-Sitzung.
- Der Handshake ist obligatorisch; jeder erste Frame, der kein JSON oder keine Verbindungsanfrage ist, führt zu einer sofortigen Trennung.
- Ereignisse werden nicht erneut wiedergegeben; Clients müssen bei Lücken ihre Daten aktualisieren.

## Verwandte Themen

- [Agentenschleife](/de/concepts/agent-loop) — detaillierter Ausführungszyklus des Agenten
- [Gateway-Protokoll](/de/gateway/protocol) — WebSocket-Protokollvertrag
- [Warteschlange](/de/concepts/queue) — Befehlswarteschlange und Parallelität
- [Sicherheit](/de/gateway/security) — Vertrauensmodell und Härtung
