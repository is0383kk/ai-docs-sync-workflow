---
read_when:
    - Live-status debuggen op de pagina Apparaten van de Control UI
    - Dubbele of verouderde instantieregels onderzoeken
    - Gateway-WS-verbinding of bakens voor systeemgebeurtenissen wijzigen
summary: Hoe OpenClaw-aanwezigheidsvermeldingen worden geproduceerd, samengevoegd en weergegeven
title: Aanwezigheid
x-i18n:
    generated_at: "2026-07-27T05:08:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac5800eebddb82e69a7d0c06733e6a19addbc57be7776e7361411866af0c60f5
    source_path: concepts/presence.md
    workflow: 16
---

OpenClaw-"presence" is een lichtgewicht overzicht op basis van best effort van:

- de **Gateway** zelf, en
- **voor gebruikers zichtbare clients die met de Gateway zijn verbonden** (Mac-app, WebChat, nodes enz.)

Presence toont live verbindingsmetadata op de pagina **Devices** van de Control UI
(onder **Settings → Devices**) en op het tabblad **Instances** van de macOS-app.

Deze pagina behandelt het clientoverzicht van de Gateway. Zie
[Presence van actieve computer](/nl/nodes/presence) om de Mac te detecteren die je het laatst
hebt gebruikt en nodewaarschuwingen daarheen te routeren.

## Presence-velden (wat wordt weergegeven)

Presence-vermeldingen zijn gestructureerde objecten met velden zoals:

- `instanceId` (optioneel, maar sterk aanbevolen): stabiele clientidentiteit (meestal `connect.client.instanceId`)
- `host`: gebruiksvriendelijke hostnaam
- `ip`: IP-adres op basis van best effort
- `version`: tekenreeks met de clientversie
- `deviceFamily` / `modelIdentifier`: hardwareaanwijzingen
- `mode`: `ui`, `webchat`, `cli`, `backend`, `node`, `probe`, `test`
- `lastInputSeconds`: seconden sinds de laatste gebruikersinvoer, indien bekend
- `reason`: vrije, door de client aangeleverde tekenreeks; de Gateway zelf genereert alleen `self`, `connect` en `disconnect`
- `deviceId`, `roles`, `scopes`: apparaatidentiteit en aanwijzingen voor rol/bereik uit de verbindingshandshake
- `ts`: tijdstempel van de laatste update (ms sinds epoch)

## Producenten (waar presence vandaan komt)

Presence-vermeldingen worden door meerdere bronnen geproduceerd en **samengevoegd**.

### 1) Eigen vermelding van de Gateway

De Gateway maakt bij het opstarten altijd een eigen vermelding aan, zodat gebruikersinterfaces
de Gateway-host tonen voordat er clients verbinding maken.

### 2) WebSocket-verbinding

Elke WS-client begint met een `connect`-verzoek. Na een geslaagde handshake
voegt de Gateway een presence-vermelding voor die verbinding toe of werkt deze bij.

#### Waarom tijdelijke control-plane-verbindingen niet worden weergegeven

CLI-opdrachten, backend-RPC-clients en probes maken vaak kortstondig verbinding. Om te voorkomen
dat deze wisselingen gedurende de volledige presence-TTL worden bewaard, worden clients in de modus
`cli`, `backend` of `probe` **niet** omgezet in presence-vermeldingen.
Clients in testmodus blijven gevolgd, omdat testsuites deze gebruiken als vervanging voor echte clients.

### 3) `system-event`-bakens

Clients kunnen uitgebreidere periodieke bakens verzenden via de methode `system-event`. De Mac-app
gebruikt dit om hostnaam, IP, versie en metadata over beschikbaarheid te rapporteren. Activiteit van
fysieke invoer maakt geen deel uit van dit generieke baken; de speciaal daarvoor bestemde native
nodegebeurtenis die wordt beschreven in [Presence van actieve computer](/nl/nodes/presence), beheert dit. De
Mac voorziet deze bakens van `system-presence-clear-last-input`; huidige Gateways gebruiken
deze achterwaarts compatibele markering om recentheid van invoer te verwijderen die door een
oudere app is bewaard. Het baken bevat ook een vaste waarde van 30 dagen, zodat oudere Gateways die
de markering negeren de exacte recentheid overschrijven in plaats van deze te bewaren. Voor deze
compatibiliteitswaarde wordt geen nieuwe activiteit gemeten.

### 4) Nodeverbindingen (rol: node)

Wanneer een node via de Gateway-WebSocket verbinding maakt met `role: node`, voegt de Gateway
een presence-vermelding voor die node toe of werkt deze bij (dezelfde flow als voor andere WS-clients).

## Regels voor samenvoegen en ontdubbelen (waarom `instanceId` belangrijk is)

Presence-vermeldingen worden opgeslagen in één in-memory map, zonder onderscheid tussen hoofdletters
en kleine letters geïndexeerd op de eerste beschikbare waarde, in deze volgorde: een gekoppelde apparaat-id,
`connect.client.instanceId` of, als laatste redmiddel, de id per verbinding.

Tijdelijke control-plane-clients worden volledig uitgesloten van tracking (zie
hierboven), zodat hun verbindings-id's nooit sleutels worden. Voor elke andere client betekent de
terugval op de verbindings-id dat een client die opnieuw verbinding maakt zonder een stabiele
`instanceId`, als een **dubbele** rij wordt weergegeven.

## TTL en begrensde grootte

Presence is bewust tijdelijk:

- **TTL:** vermeldingen ouder dan 5 minuten worden verwijderd
- **Maximumaantal vermeldingen:** 200 (oudste eerst verwijderd)

Hierdoor blijft de lijst actueel en wordt onbeperkte groei van het geheugengebruik voorkomen.

## Aandachtspunt bij externe verbindingen/tunnels (loopback-IP's)

Wanneer een client verbinding maakt via een SSH-tunnel/lokale poortdoorsturing, kan de Gateway
het externe adres zien als `127.0.0.1`. Om te voorkomen dat dit tunneladres
als IP-adres van de client wordt opgeslagen, laat de verbindingsafhandeling `ip` volledig
weg voor gedetecteerde lokale clients (loopback), in plaats van het loopback-adres
in de vermelding te schrijven.

## Consumenten

### Pagina Devices van de Control UI

De pagina **Devices** combineert `system-presence` met duurzame koppelings- en
noderecords. Deze zet het eigen Gateway-baken bovenaan vast en gebruikt overeenkomende apparaat- of
instantie-id's voor live metadata over platform, versie, model en recentheid van invoer.

### Tabblad Instances van macOS

De macOS-app geeft de uitvoer van `system-presence` weer en past een kleine statusindicator
(Active/Idle/Stale) toe op basis van de ouderdom van de laatste update.

## Tips voor foutopsporing

- Roep `system-presence` aan op de Gateway om de onbewerkte lijst te bekijken.
- Als je dubbele vermeldingen ziet:
  - controleer of clients tijdens de handshake een stabiele `client.instanceId` verzenden
  - controleer of periodieke bakens dezelfde `instanceId` gebruiken
  - controleer of `instanceId` ontbreekt in de van de verbinding afgeleide vermelding (dubbele vermeldingen zijn dan te verwachten)

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Presence van actieve computer" href="/nl/nodes/presence" icon="computer-mouse">
    Hoe fysieke invoer op een Mac een actieve node selecteert en verbindingswaarschuwingen routeert.
  </Card>
  <Card title="Typindicatoren" href="/nl/concepts/typing-indicators" icon="ellipsis">
    Wanneer typindicatoren worden verzonden en hoe je ze afstemt.
  </Card>
  <Card title="Streaming en segmentering" href="/nl/concepts/streaming" icon="bars-staggered">
    Uitgaande streaming, segmentering en opmaak per kanaal.
  </Card>
  <Card title="Gateway-architectuur" href="/nl/concepts/architecture" icon="diagram-project">
    Gateway-componenten en het WebSocket-protocol dat presence-updates aanstuurt.
  </Card>
  <Card title="Gateway-protocol" href="/nl/gateway/protocol" icon="plug">
    Het wire-protocol voor `connect`, `system-event` en `system-presence`.
  </Card>
</CardGroup>
