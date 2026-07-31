---
read_when:
    - Je bouwt een externe app, script, dashboard, CI-taak of IDE-extensie die met OpenClaw communiceert
    - Je kiest tussen Gateway-RPC en de Plugin-SDK
    - Je integreert met Gateway-agentruns, sessies, gebeurtenissen, goedkeuringen, modellen of tools
    - Je koppelt een hostingcontroller aan een externe activeringsplanner
sidebarTitle: External apps
summary: Huidig integratiepad voor externe apps, scripts, dashboards, CI-taken en IDE-extensies
title: Gateway-integraties voor externe apps
x-i18n:
    generated_at: "2026-07-27T05:04:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 276c6f4173197683a60770327e131e6ab2fa4d33f416ba96c170539df7246f83
    source_path: gateway/external-apps.md
    workflow: 16
---

Externe apps communiceren met OpenClaw via het Gateway-protocol: WebSocket-
transport plus RPC-methoden. Gebruik dit wanneer een script, dashboard, CI-taak, IDE-
extensie of ander proces agentruns wil starten, gebeurtenissen wil streamen, op
resultaten wil wachten, werk wil annuleren of Gateway-resources wil inspecteren.

<Note>
  Begin voor npm-pakketten, apparaatkoppeling, herstel van de verbinding, geschiedenis, abonnementen
  en goedkeuringen met
  [Een Gateway-client bouwen](https://docs.openclaw.ai/gateway/clients). Als jouw
  app de Gateway als een onderliggend proces beheert, lees dan ook
  [OpenClaw insluiten](https://docs.openclaw.ai/gateway/embedding). Tijdens de
  eerste pakketuitrol kan npm `E404` retourneren totdat de eerste OpenClaw-release
  met pakketten is gepubliceerd.
</Note>

<Note>
  Deze pagina is bedoeld voor code buiten het OpenClaw-proces. Plugincode die
  binnen OpenClaw wordt uitgevoerd, moet in plaats daarvan gedocumenteerde `openclaw/plugin-sdk/*`-subpaden gebruiken.
</Note>

## Wat vandaag beschikbaar is

| Oppervlak                                                        | Status        | Gebruik dit voor                                                                               |
| ---------------------------------------------------------------- | ------------- | ---------------------------------------------------------------------------------------------- |
| [Gateway-clienthandleiding](https://docs.openclaw.ai/gateway/clients) | Releasetraject | npm-pakketten, authenticatie, opnieuw verbinden, geschiedenis, gebeurtenissen, goedkeuringen en versiebeleid. |
| [Handleiding voor insluiten](https://docs.openclaw.ai/gateway/embedding) | Releasetraject | Omgeving van onderliggende processen, gereedheid, levenscyclus, herstel, RPC-eigenaarschap en verpakking. |
| [Gateway-protocol](/nl/gateway/protocol)                            | Gereed        | WebSocket-transport, verbindingshandshake, authenticatiebereiken, protocolversiebeheer en gebeurtenissen. |
| [Gateway RPC-referentie](/nl/reference/rpc)                         | Gereed        | Huidige Gateway-methoden voor agents, sessies, taken, modellen, tools, artefacten en goedkeuringen. |
| [`openclaw agent`](/nl/cli/agent)                                 | Gereed        | Eenmalige scriptintegratie wanneer het aanroepen van de CLI via de shell volstaat.             |
| [`openclaw message`](/nl/cli/message)                               | Gereed        | Berichten of kanaalacties vanuit scripts verzenden.                                            |

## Aanbevolen werkwijze

1. Voer een Gateway uit of detecteer er een.
2. Maak verbinding via het [Gateway-protocol](/nl/gateway/protocol).
3. Roep gedocumenteerde RPC-methoden aan uit de [Gateway RPC-referentie](/nl/reference/rpc).
4. Zet de OpenClaw-versie waartegen je test vast.
5. Controleer de RPC-referentie opnieuw wanneer je OpenClaw bijwerkt.

Begin voor agentruns met de RPC `agent` en combineer deze met `agent.wait` voor een
eindresultaat. Gebruik de `sessions.*`-methoden voor duurzame gespreksstatus.
Abonneer je voor UI-integraties op Gateway-gebeurtenissen en geef alleen de
gebeurtenisfamilies weer die jouw app begrijpt.

## Coöperatieve opschorting door de host

Hostingcontrollers die een actief proces bevriezen of er een snapshot van maken, kunnen de
hostneutrale opschortingshandshake gebruiken:

1. Sta geen nieuwe externe toegang meer toe die door de host wordt beheerd.
2. Roep `gateway.suspend.prepare` aan met een stabiele, unieke `requestId`.
3. Als het antwoord `busy` is, laat je het proces actief en probeer je het later opnieuw.
4. Als het `ready` is, sla je de geretourneerde `suspensionId` op en bevriest het proces of maakt
   er een snapshot van vóór `expiresAtMs`.
5. Roep na het hervatten, of als de opschorting wordt afgebroken, `gateway.suspend.resume` aan
   met die `suspensionId` via de bestaande WebSocket of het Admin HTTP-
   besturingspad.

Een voorbereide Gateway weigert nieuwe WebSocket-handshakes. Een WebSocket-controller
moet zijn geauthenticeerde verbinding tijdens de hostbewerking openhouden. Als dit
niet kan worden gegarandeerd, schakel dan vóór de voorbereiding de
[Admin HTTP RPC-plugin](/nl/plugins/admin-http-rpc) in en gebruik deze. Als het
besturingspad verloren gaat, wacht dan tot de lease van twee minuten verloopt voordat
je opnieuw verbinding maakt; na het verlopen wordt toegang automatisch weer toegestaan.

Het RPC-contract is:

- `gateway.suspend.prepare` — `operator.admin`; parameters
  `{ "requestId": "stable-host-operation-id" }`
- `gateway.suspend.status` — `operator.read`; parameters
  `{ "suspensionId": "id-from-prepare" }`
- `gateway.suspend.resume` — `operator.admin`; parameters
  `{ "suspensionId": "id-from-prepare" }`

ID's worden bijgesneden, moeten een teken bevatten dat geen witruimte is en zijn beperkt tot
128 tekens. Een bezet voorbereidingsresultaat bevat `status: "busy"`, `reason`,
`retryAfterMs`, `activeCount` en `blockers`. Een gereed resultaat heeft deze vorm:

```json
{
  "status": "ready",
  "suspensionId": "2c3f...",
  "expiresAtMs": 1770000000000,
  "activeCount": 0,
  "blockers": []
}
```

Status retourneert `{"status":"running"}` of een gereed resultaat met `expiresAtMs`.
Hervatten retourneert `{"ok":true,"status":"running","resumed":true}`; als je dit
na een geslaagde hervatting herhaalt, wordt `resumed: false` geretourneerd.

Een concurrerend aanvraag-ID of tijdelijke fout bij het hervatten van de planner retourneert de opnieuw
te proberen fout `UNAVAILABLE` met `retryAfterMs`. Tijdens het herstel van de planner retourneren voorbereiding, status
en hervatten allemaal die fout, blijft de Gateway niet gereed en
gesloten bij fouten, en mag de host deze niet bevriezen of er een snapshot van maken. OpenClaw probeert
de planner automatisch opnieuw en staat pas weer toegang toe nadat het herstel is geslaagd. Een
niet-overeenkomend hervattings-ID retourneert `INVALID_REQUEST`. Voorbereiding deelt het
schrijfbudget van het Gateway-besturingsvlak van drie pogingen per minuut; respecteer de geretourneerde
wachttijd voor een nieuwe poging. WebSocket-clients worden per apparaat en IP gegroepeerd. Admin HTTP-
controllers worden per vastgesteld client-IP gegroepeerd, waardoor controllers achter één
proxy een budget kunnen delen.

Voorbereiding kan alleen weigeren: OpenClaw sluit nieuwe toegang voor root/sessie/opdracht,
pauzeert automatische Cron-ticks en inspecteert werk synchroon. Als er iets
actief is, hervat het de planner en staat het opnieuw toegang toe voordat
`busy` wordt geretourneerd; dat werk wordt niet onderbroken of afgehandeld. Een gereedheidslease duurt twee
minuten. Als `prepare` met dezelfde `requestId` wordt herhaald, wordt deze verlengd; bij het verlopen
wordt de planner hervat voordat toegang opnieuw wordt toegestaan.
Een herstartsignaal dat tijdens een gereedheidslease moet worden verzonden, wacht totdat de lease
wordt hervat; een lopende herstart zorgt ervoor dat voorbereiding `busy` retourneert.

Terwijl de Gateway gereed is, blijft `/healthz` actief en retourneert `/readyz` `503`. Lokale of
geauthenticeerde gereedheidsantwoorden bevatten `gateway-draining`; niet-geauthenticeerde
externe controles ontvangen alleen `{ "ready": false }`. De HTTP-statuscontrole,
opschortingsmethoden op bestaande WebSocket-verbindingen en een reeds ingeschakelde
Admin HTTP RPC-route blijven beschikbaar. Andere RPC's retourneren de opnieuw te proberen fout
`UNAVAILABLE`. Ingebouwde HTTP-routes voor gebruikerswerk en gewone HTTP-routes van plugins,
waaronder OpenAI-compatibele API's, tool-/sessiebewerkingen, Node-observaties en
geconfigureerde hooks, retourneren `503` met `error.code: "gateway_unavailable"`. Nieuwe
WebSocket-upgrades die eigendom zijn van plugins retourneren ook `503`; dit betreft het eigenaarschap
van de upgrade, niet werk dat later via een bestaande pluginsocket wordt uitgevoerd.

Deze handshake bewaart geen inkomende berichten, stopt geen kanaaltransporten
van derden en bestuurt het hostingplatform niet. De host moet vóór de voorbereiding
de eigen toegang afschermen en blijft verantwoordelijk voor activeren, snapshots/bevriezen en
stoppen. `activeCount` is het totale aantal bijgehouden werkzaamheden, terwijl `blockers`
de categorietellingen die niet nul zijn en begrensde taakdetails bevat. Dit is geen
algemene barrière voor procesinactiviteit. Een `background-exec`-blokkering bevat alleen
geaggregeerde gegevens: opdrachttekst, proces-ID's, uitvoer en sessie- of bereik-ID's worden nooit
via het protocol doorgegeven. Kanaalstatus, onderhoud, cacheverversing, bestaande
WebSocket-sessies van plugins en niet-geregistreerd achtergrondwerk dat eigendom is van plugins kunnen
actief blijven.
Het hostingplatform moet de volledige processtructuur en het bijbehorende
bestandssysteem consistent bevriezen of er een snapshot van maken; met dit eerste
contract kan niet worden bewezen dat niet-geregistreerd werk inactief is.

<Tip>
  Houd voor de planning van het activeren door de host het OpenClaw-gerichte deel in een in-process
  Plugin en projecteer idempotente volledige snapshots naar de externe hostadapter.
  De hostingcontroller mag de Plugin SDK niet importeren en de Cron-
  status niet reconstrueren uit gebeurtenisdelta's. Zie [Veilige externe Cron-
  projectie](/nl/plugins/hooks#safe-external-cron-projection).
</Tip>

## Appcode versus plugincode

Gebruik Gateway RPC wanneer code buiten OpenClaw wordt uitgevoerd:

- Node-scripts die agentruns starten of observeren
- CI-taken die een Gateway aanroepen
- dashboards en beheerpanelen
- IDE-extensies
- externe bridges die geen kanaalplugins hoeven te worden
- integratietests met gesimuleerde of echte Gateway-transporten

Gebruik de Plugin SDK wanneer code binnen OpenClaw wordt uitgevoerd:

- providerplugins
- kanaalplugins
- tool- of levenscyclushooks
- agentharnasplugins
- vertrouwde runtimehelpers

Externe apps mogen `openclaw/plugin-sdk/*` niet importeren; deze subpaden zijn bestemd voor
plugins die door OpenClaw worden geladen.

## Gerelateerd

- [Een Gateway-client bouwen](https://docs.openclaw.ai/gateway/clients)
- [OpenClaw insluiten](https://docs.openclaw.ai/gateway/embedding)
- [Gateway-protocol](/nl/gateway/protocol)
- [Gateway RPC-referentie](/nl/reference/rpc)
- [CLI-agentopdracht](/nl/cli/agent)
- [CLI-berichtopdracht](/nl/cli/message)
- [Agentlus](/nl/concepts/agent-loop)
- [Agentruntimes](/nl/concepts/agent-runtimes)
- [Sessies](/nl/concepts/session)
- [Achtergrondtaken](/nl/automation/tasks)
- [ACP-agents](/nl/tools/acp-agents)
- [Overzicht van de Plugin SDK](/nl/plugins/sdk-overview)
