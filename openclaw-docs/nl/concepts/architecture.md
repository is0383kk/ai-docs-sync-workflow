---
read_when:
    - Werken aan het Gateway-protocol, clients of transportlagen
summary: WebSocket-Gatewayarchitectuur, componenten en clientstromen
title: Gateway-architectuur
x-i18n:
    generated_at: "2026-07-27T04:56:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8054bd87f738b957c24f8d6965d55365de2293d44902530a9ba778afa597cc7
    source_path: concepts/architecture.md
    workflow: 16
---

## Overzicht

- Eén lang actieve **Gateway** beheert alle berichteninterfaces (WhatsApp via
  Baileys, Telegram via grammY, Slack, Discord, Signal, iMessage, WebChat).
- Clients op het besturingsvlak (macOS-app, CLI, webinterface, automatiseringen) maken
  via **WebSocket** verbinding met de Gateway op de geconfigureerde bindhost (standaard
  `127.0.0.1:18789`).
- **Nodes** (macOS/iOS/Android/headless) maken ook via **WebSocket** verbinding, maar
  declareren `role: node` met expliciete mogelijkheden/opdrachten.
- Eén Gateway per host; dit is de enige plek die een WhatsApp-sessie opent.
- De **canvashost** wordt aangeboden door de HTTP-server van de Gateway onder:
  - `/__openclaw__/canvas/` (door de agent bewerkbare HTML/CSS/JS)
  - `/__openclaw__/a2ui/` (A2UI-host)

  Deze gebruikt dezelfde poort als de Gateway (standaard `18789`).

## Componenten en flows

### Gateway (daemon)

- Onderhoudt providerverbindingen.
- Biedt een getypeerde WS-API (verzoeken, antwoorden, door de server gepushte gebeurtenissen).
- Valideert inkomende frames aan de hand van JSON Schema.
- Verzendt gebeurtenissen zoals `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`.

### Clients (Mac-app / CLI / webbeheer)

- Eén WS-verbinding per client.
- Versturen verzoeken (`health`, `status`, `send`, `agent`, `system-presence`).
- Abonneren zich op gebeurtenissen (`tick`, `agent`, `presence`, `shutdown`).

### Nodes (macOS / iOS / Android / headless)

- Maken verbinding met **dezelfde WS-server** met `role: node`.
- Geven een apparaatidentiteit op in `connect`; koppeling is **apparaatgebaseerd** (rol `node`) en
  goedkeuring wordt opgeslagen in het opslagmedium voor apparaatkoppelingen.
- Stellen opdrachten beschikbaar zoals `canvas.*`, `camera.*`, `screen.record`, `location.get`.

Protocoldetails: [Gateway-protocol](/nl/gateway/protocol)

### WebChat

- Statische gebruikersinterface die de WS-API van de Gateway gebruikt voor chatgeschiedenis en het verzenden van berichten.
- Maakt in externe configuraties verbinding via dezelfde SSH-/Tailscale-tunnel als andere
  clients.

## Levenscyclus van de verbinding (één client)

```mermaid
sequenceDiagram
    participant Client
    participant Gateway

    Client->>Gateway: req:connect
    Gateway-->>Client: res (ok)
    Note right of Gateway: of res-fout + sluiten
    Note left of Client: payload=hello-ok<br>momentopname: aanwezigheid + status

    Gateway-->>Client: event:presence
    Gateway-->>Client: event:tick

    Client->>Gateway: req:agent
    Gateway-->>Client: res:agent<br>bevestiging {runId, status:"accepted"}
    Gateway-->>Client: event:agent<br>(streaming)
    Gateway-->>Client: res:agent<br>definitief {runId, status, summary}
```

## Wire-protocol (samenvatting)

- Transport: WebSocket, tekstframes met JSON-payloads.
- Het eerste frame **moet** `connect` zijn.
- Na de handshake:
  - Verzoeken: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - Gebeurtenissen: `{type:"event", event, payload, seq?, stateVersion?}`
- `hello-ok.features.methods` / `events` zijn detectiemetadata, geen
  gegenereerde dump van elke aanroepbare hulproute.
- Authenticatie met een gedeeld geheim gebruikt `connect.params.auth.token` of
  `connect.params.auth.password`, afhankelijk van de geconfigureerde authenticatiemodus van de Gateway.
- Modi die identiteit bevatten, zoals Tailscale Serve
  (`gateway.auth.allowTailscale: true`) of niet-loopback
  `gateway.auth.mode: "trusted-proxy"`, voldoen aan de authenticatie via verzoekheaders
  in plaats van `connect.params.auth.*`.
- `gateway.auth.mode: "none"` voor privé-ingang schakelt authenticatie met een gedeeld geheim
  volledig uit; houd die modus uitgeschakeld voor openbare/niet-vertrouwde ingangen.
- Idempotentiesleutels zijn vereist voor methoden met neveneffecten (`send`, `agent`) om
  veilig opnieuw te proberen; de server houdt een kortlevende deduplicatiecache bij.
- Nodes moeten `role: "node"` plus mogelijkheden/opdrachten/machtigingen opnemen in `connect`.

## Koppeling en lokaal vertrouwen

- Alle WS-clients (operators + Nodes) nemen een **apparaatidentiteit** op in `connect`.
- Nieuwe apparaat-ID's vereisen goedkeuring van de koppeling; de Gateway verstrekt een **apparaattoken**
  voor volgende verbindingen.
- Directe lokale loopbackverbindingen kunnen automatisch worden goedgekeurd om de gebruikerservaring
  op dezelfde host soepel te houden.
- OpenClaw heeft ook een beperkt zelfverbindingspad dat lokaal is voor de backend/container voor
  vertrouwde hulpflows met een gedeeld geheim.
- Verbindingen via tailnet en LAN, inclusief tailnet-binds op dezelfde host, vereisen nog steeds
  expliciete goedkeuring van de koppeling.
- Alle verbindingen moeten de nonce `connect.challenge` ondertekenen. De ondertekeningspayload `v3`
  bindt ook `platform` en `deviceFamily`; de Gateway zet gekoppelde metadata vast bij
  opnieuw verbinden en vereist een herstelkoppeling voor wijzigingen in metadata.
- **Niet-lokale** verbindingen vereisen nog steeds expliciete goedkeuring.
- Gateway-authenticatie (`gateway.auth.*`) is nog steeds van toepassing op **alle** verbindingen, lokaal of
  extern.

Details: [Gateway-protocol](/nl/gateway/protocol), [Koppeling](/nl/channels/pairing),
[Beveiliging](/nl/gateway/security).

## Protocoltypering en codegeneratie

- TypeBox-schema's definiëren het protocol.
- JSON Schema wordt vanuit die schema's gegenereerd.
- Swift-modellen worden vanuit JSON Schema gegenereerd.

## Externe toegang

- Voorkeur: Tailscale of VPN.
- Alternatief: SSH-tunnel

  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
  ```

- Dezelfde handshake en hetzelfde authenticatietoken zijn van toepassing via de tunnel.
- TLS en optionele pinning kunnen voor WS worden ingeschakeld in externe configuraties.

## Momentopname van de werking

- Starten: `openclaw gateway` (op de voorgrond, logt naar stdout).
- Status: `health` via WS (ook opgenomen in `hello-ok`).
- Toezicht: launchd/systemd voor automatisch opnieuw starten.

## Invarianten

- Precies één Gateway beheert één Baileys-sessie per host.
- De handshake is verplicht; elk eerste frame dat geen JSON of geen connect-frame is, leidt tot een harde sluiting.
- Gebeurtenissen worden niet opnieuw afgespeeld; clients moeten vernieuwen bij hiaten.

## Gerelateerd

- [Agentlus](/nl/concepts/agent-loop) — gedetailleerde uitvoeringscyclus van de agent
- [Gateway-protocol](/nl/gateway/protocol) — contract van het WebSocket-protocol
- [Wachtrij](/nl/concepts/queue) — opdrachtenwachtrij en gelijktijdigheid
- [Beveiliging](/nl/gateway/security) — vertrouwensmodel en versterking
