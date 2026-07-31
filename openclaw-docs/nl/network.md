---
read_when:
    - Je hebt het overzicht van de netwerkarchitectuur en beveiliging nodig
    - Je lost problemen op met lokale toegang versus toegang via het tailnet of met koppelen
    - Je wilt de canonieke lijst met netwerkdocumentatie
summary: 'Netwerkhub: Gateway-oppervlakken, koppeling, detectie en beveiliging'
title: Netwerk
x-i18n:
    generated_at: "2026-07-27T06:20:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9751bb0fe71009455b243b109ef7ef4eda08d58f940f7dcef305800a5ed89586
    source_path: network.md
    workflow: 16
---

Deze hub bevat links naar de kerndocumentatie over hoe OpenClaw verbinding maakt met apparaten, ze koppelt en beveiligt
via localhost, LAN en tailnet.

## Kernmodel

De meeste bewerkingen verlopen via de Gateway (`openclaw gateway`), één langlopend proces dat de kanaalverbindingen en het WebSocket-besturingsvlak beheert.

- **Eerst loopback**: de Gateway-WS gebruikt standaard `ws://127.0.0.1:18789`.
  Bindingen buiten loopback weigeren te starten zonder een geldig authenticatiepad voor de Gateway:
  authenticatie met een gedeeld geheim token/wachtwoord, of een correct geconfigureerde
  `trusted-proxy`-implementatie buiten loopback.
- **Eén Gateway per host** wordt aanbevolen. Voer voor isolatie meerdere Gateways uit met geïsoleerde profielen en poorten ([Meerdere Gateways](/nl/gateway/multiple-gateways)).
- **Canvas-host** wordt aangeboden op dezelfde poort als de Gateway (`/__openclaw__/canvas/`, `/__openclaw__/a2ui/`) en wordt door Gateway-authenticatie beschermd wanneer deze buiten loopback is gebonden.
- **Externe toegang** verloopt doorgaans via een SSH-tunnel of Tailscale-VPN ([Externe toegang](/nl/gateway/remote)).

Belangrijke naslaginformatie:

- [Gateway-architectuur](/nl/concepts/architecture)
- [Gateway-protocol](/nl/gateway/protocol)
- [Gateway-runbook](/nl/gateway)
- [Webinterfaces en bindingsmodi](/nl/web)

## Koppeling en identiteit

- [Overzicht van koppeling (DM + Nodes)](/nl/channels/pairing)
- [Door de Gateway beheerde Node-koppeling](/nl/gateway/pairing)
- [Apparaten-CLI (koppeling + tokenrotatie)](/nl/cli/devices)
- [Koppelings-CLI (DM-goedkeuringen)](/nl/cli/pairing)

Lokaal vertrouwen:

- Directe lokale loopbackverbindingen (zonder doorgestuurde/proxyheaders) kunnen
  automatisch worden goedgekeurd voor koppeling, zodat de gebruikerservaring op dezelfde host soepel blijft.
- OpenClaw heeft ook een beperkt zelfverbindingspad dat lokaal is voor de backend/container
  voor vertrouwde helperflows met een gedeeld geheim.
- Voor tailnet- en LAN-clients, inclusief tailnet-bindingen op dezelfde host, blijft
  expliciete goedkeuring van de koppeling vereist.

## Detectie en transporten

- [Detectie en transporten](/nl/gateway/discovery)
- [Bonjour / mDNS](/nl/gateway/bonjour)
- [Externe toegang (SSH)](/nl/gateway/remote)
- [Tailscale](/nl/gateway/tailscale)

## Nodes en transporten

- [Overzicht van Nodes](/nl/nodes)
- [Bridge-protocol (verouderde Nodes, historisch)](/nl/gateway/bridge-protocol)
- [Node-runbook: iOS](/nl/platforms/ios)
- [Node-runbook: Android](/nl/platforms/android)

## Beveiliging

- [Beveiligingsoverzicht](/nl/gateway/security)
- [Naslaginformatie voor Gateway-configuratie](/nl/gateway/configuration)
- [Probleemoplossing](/nl/gateway/troubleshooting)
- [Doctor](/nl/gateway/doctor)

## Gerelateerd

- [Gateway-runbook](/nl/gateway)
- [Externe toegang](/nl/gateway/remote)
