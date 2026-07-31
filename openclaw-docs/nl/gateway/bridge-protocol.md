---
read_when:
    - Onderzoek van oude Node-clientcode of gearchiveerde koppelingslogboeken
    - Controleren wat de verouderde Node-interface voorheen beschikbaar stelde
summary: 'Historisch bridgeprotocol (verouderde nodes): TCP JSONL, koppeling, bereikgebonden RPC'
title: Bridgeprotocol
x-i18n:
    generated_at: "2026-07-27T05:04:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e8b69c59f2170439f0e7b139bf5bbdb429d7c9d8dde7b36cd64aab63939c95d
    source_path: gateway/bridge-protocol.md
    workflow: 16
---

<Warning>
De TCP-bridge is **verwijderd**. Huidige OpenClaw-builds bevatten de bridge-listener niet en de configuratiesleutels `bridge.*` maken geen deel meer uit van het schema. Deze pagina dient uitsluitend als historische referentie. Gebruik het [Gateway-protocol](/nl/gateway/protocol) voor alle node-/operatorclients.
</Warning>

## Waarom deze bestond

- **Beveiligingsgrens**: stelde een kleine toelatingslijst beschikbaar in plaats van het volledige API-oppervlak van de Gateway.
- **Koppeling + node-identiteit**: toelating van nodes werd beheerd door de Gateway en was gekoppeld aan een token per node.
- **Detectie-UX**: nodes konden Gateways via Bonjour op het LAN detecteren of rechtstreeks via een tailnet verbinding maken.
- **Loopback-WS**: het volledige WS-besturingsvlak bleef lokaal, tenzij het via SSH werd getunneld.

## Transport

- TCP, één JSON-object per regel (JSONL).
- Optionele TLS (`bridge.tls.enabled: true`).
- De standaardlistenerpoort was `18790`.

Wanneer TLS was ingeschakeld, bevatten TXT-records voor detectie `bridgeTls=1` plus `bridgeTlsSha256` als niet-geheime aanwijzing. Bonjour-/mDNS-TXT-records zijn niet geauthenticeerd; clients konden de geadverteerde vingerafdruk zonder andere verificatie buiten het communicatiekanaal niet als gezaghebbende pin beschouwen.

## Handshake en koppeling

1. De client verzendt `hello` met nodemetadata plus een token (indien al gekoppeld).
2. Indien niet gekoppeld, antwoordt de Gateway met `error` (`NOT_PAIRED` / `UNAUTHORIZED`).
3. De client verzendt `pair-request`.
4. De Gateway wacht op goedkeuring en verzendt vervolgens `pair-ok` en `hello-ok`.

`hello-ok` retourneerde voorheen `serverName`; gehoste Plugin-oppervlakken worden nu via `pluginSurfaceUrls` in het huidige Gateway-protocol aangekondigd (Canvas/A2UI gebruikt `pluginSurfaceUrls.canvas`).

## Frames

Van client naar Gateway:

- `req` / `res`: afgebakende Gateway-RPC (chat, sessies, configuratie, status, voicewake, skills.bins).
- `event`: nodesignalen (spraaktranscript, agentverzoek, chatabonnement, uitvoeringslevenscyclus).

Van Gateway naar client:

- `invoke` / `invoke-res`: nodeopdrachten (`canvas.*`, `camera.*`, `screen.record`, `location.get`, `sms.send`).
- `event`: chatupdates voor sessies waarop een abonnement bestond.
- `ping` / `pong`: keepalive.

De handhaving van de toelatingslijst bevond zich in `src/gateway/server-bridge.ts` (verwijderd).

## Gebeurtenissen in de uitvoeringslevenscyclus

Nodes verzonden `exec.finished` om voltooide `system.run`-activiteit beschikbaar te stellen, die door de Gateway aan systeemgebeurtenissen werd gekoppeld (verouderde nodes konden ook `exec.started` verzenden). `exec.denied` markeerde een geweigerde `system.run`-poging als definitieve weigering zonder een systeemgebeurtenis in de wachtrij te plaatsen of agentwerk te activeren.

Payloadvelden (allemaal optioneel, tenzij anders vermeld):

| Veld                             | Opmerkingen                                                                                    |
| -------------------------------- | ---------------------------------------------------------------------------------------------- |
| `sessionKey`                     | Vereist. Agentsessie voor gebeurteniscorrelatie en, voor `exec.finished`, aflevering van systeemgebeurtenissen. |
| `runId`                          | Unieke uitvoerings-id voor groepering.                                                         |
| `command`                        | Onbewerkte of opgemaakte opdrachttekenreeks.                                                   |
| `exitCode`, `timedOut`, `output` | Voltooiingsdetails (alleen indien voltooid).                                                   |
| `reason`                         | Reden voor weigering (alleen indien geweigerd).                                                |

## Historisch tailnet-gebruik

- Koppel de bridge aan een tailnet-IP: `bridge.bind: "tailnet"` in `~/.openclaw/openclaw.json` (alleen historisch; `bridge.*` is niet langer een geldige configuratie).
- Clients maakten verbinding via een MagicDNS-naam of tailnet-IP.
- Bonjour werkt niet over verschillende netwerken; anders was wide-area DNS-SD of een handmatig opgegeven host/poort vereist.

## Versiebeheer

De bridge was impliciet v1, zonder onderhandeling over een minimum- en maximumversie. Huidige node-/operatorclients gebruiken het WebSocket-[Gateway-protocol](/nl/gateway/protocol), dat wel over een bereik van protocolversies onderhandelt.

## Gerelateerd

- [Gateway-protocol](/nl/gateway/protocol)
- [Nodes](/nl/nodes)
