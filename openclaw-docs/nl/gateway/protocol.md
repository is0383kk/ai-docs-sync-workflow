---
read_when:
    - Gateway-WS-clients implementeren of bijwerken
    - Fouten opsporen bij protocolverschillen of verbindingsproblemen
    - Protocolschema/-modellen opnieuw genereren
summary: 'Gateway-WebSocket-protocol: handshake, frames, versiebeheer'
title: Gateway-protocol
x-i18n:
    generated_at: "2026-07-27T05:52:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89d637a9070bc6512a182fea0fd890b56287e0080515ba4fba9b2591c6247e0d
    source_path: gateway/protocol.md
    workflow: 16
---

Het Gateway-WS-protocol is het centrale besturingsvlak en Node-transport voor
OpenClaw. Operator- en Node-clients (CLI, web-UI, macOS-app, iOS-/Android-nodes,
headless nodes) maken verbinding via WebSocket en declareren tijdens de
handshake een **rol** en **scope**.

## npm-pakketten

Deze pakketten worden meegeleverd met OpenClaw-releasereeksen. Tijdens de eerste uitrol
kan npm `E404` retourneren totdat de eerste release met pakketten is gepubliceerd.

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  publiceert de schema's, validators, TypeScript-typen, lichtgewicht helpers voor frames en fouten,
  en versieconstanten. De tarball bevat het gegenereerde
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  machineleesbare contract.
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  publiceert de referentieclient voor Node en een browserveilige ingang op
  `@openclaw/gateway-client/browser`.

Zie voor richtlijnen over de levenscyclus van toepassingen
[Een Gateway-client bouwen](https://docs.openclaw.ai/gateway/clients). Zie voor apps
die de Gateway als onderliggend proces beheren
[OpenClaw insluiten](https://docs.openclaw.ai/gateway/embedding).

## Transport en framing

- WebSocket, tekstframes, JSON-payloads.
- Het eerste frame **moet** een `connect`-verzoek zijn.
- Frames vóór de verbinding zijn beperkt tot 64 KiB (`MAX_PREAUTH_PAYLOAD_BYTES`). Volg na
  de handshake `hello-ok.policy.maxPayload` en
  `hello-ok.policy.maxBufferedBytes`. Als diagnostiek is ingeschakeld, genereren te grote
  binnenkomende frames en trage uitgaande buffers `payload.large`-gebeurtenissen voordat
  de Gateway het frame sluit of laat vallen. Deze gebeurtenissen bevatten `surface`, bytegroottes,
  limieten en een veilige redencode, maar nooit berichtteksten, inhoud van
  bijlagen, onbewerkte framebytes, tokens, cookies of geheimen.

Framevormen:

- Verzoek: `{type:"req", id, method, params}`
- Antwoord: `{type:"res", id, ok, payload|error}`
- Gebeurtenis: `{type:"event", event, payload, seq?, stateVersion?}`

Antwoordfouten gebruiken `{ code, message, details?, retryable?, retryAfterMs? }`.
Clients moeten vertakken op `code` en `details.code`; `message` blijft voor mensen leesbaar
en kan veranderen, behalve waar een compatibiliteitsopmerking anders aangeeft. Autorisatiefouten
op methodeniveau gebruiken `code: "FORBIDDEN"` op het hoogste niveau met gestructureerde
details over ontbrekende scopes:

- Ontbrekende scope: `{ code: "MISSING_SCOPE", missingScope, requiredScopes }`.
  `requiredScopes` is de volledige bekende scopeset voor de aangevraagde bewerking.
  Het verouderde `missing scope: <scope>`-bericht blijft behouden voor oudere clients.

Clients moeten eerst `details` lezen en het verouderde bericht alleen als compatibiliteits-
fallback gebruiken. `readMissingScopeError` en `readMissingScopeErrorDetails` worden geëxporteerd vanuit
`@openclaw/gateway-protocol/gateway-error-details`; de browserveilige Gateway-client
exporteert ze opnieuw vanuit `@openclaw/gateway-client/browser`.

De schema's worden geëxporteerd als `GatewayErrorDetailsSchema`,
`MissingScopeErrorDetailsSchema` vanuit `@openclaw/gateway-protocol/schema`.
HTTP-scopefouten weerspiegelen het `MISSING_SCOPE`-object onder `error.details` en
gebruiken HTTP-status `403`.

Methoden met neveneffecten vereisen idempotentiesleutels (zie schema).

## Handshake

De Gateway stuurt vóór de verbinding een challenge:

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

De client antwoordt met `connect`:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

De Gateway antwoordt met `hello-ok`:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`, `features`, `snapshot`, `policy` en `auth` zijn allemaal vereist door
`HelloOkSchema` (`packages/gateway-protocol/src/schema/frames.ts`). `auth`
rapporteert de overeengekomen rol/scopes, zelfs wanneer geen apparaattoken wordt uitgegeven (vorm
hierboven). `pluginSurfaceUrls` is optioneel en koppelt namen van Plugin-oppervlakken (bijv.
`canvas`) aan gehoste URL's met scopes; deze kunnen verlopen, dus nodes roepen
`node.pluginSurface.refresh` aan met `{ "surface": "canvas" }` voor een nieuwe ingang.
Het verouderde pad `canvasHostUrl` / `canvasCapability` / `node.canvas.capability.refresh`
wordt niet ondersteund; gebruik Plugin-oppervlakken.
De optionele `appliedConfigHash` van de snapshot is de opgeloste bronconfiguratierevisie
die door de actieve Gateway-runtime is geaccepteerd. Clients kunnen deze vergelijken met
`config.get.configRevisionHash` om te bepalen of voor een nieuwere opgeslagen configuratie nog steeds
een herstart nodig is. `config.get.hash` blijft de onbewerkte revisie van het hoofdbestand die wordt gebruikt door
conflictbeveiligingen bij het schrijven van configuraties.

Terwijl de Gateway de opstart-sidecars nog voltooit, kan `connect` een
opnieuw te proberen `UNAVAILABLE`-fout retourneren met `details.reason: "startup-sidecars"` en
`retryAfterMs`. Probeer het binnen het verbindingsbudget opnieuw in plaats van dit als
een definitieve handshakefout te behandelen.

Wanneer een apparaattoken wordt uitgegeven, voegt `hello-ok.auth` dit toe:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

De ingebouwde bootstrap via QR-/installatiecode is een overdrachtspad voor mobiele apparaten. Een geslaagde
basisverbinding met een installatiecode retourneert een primair Node-token plus één begrensd
operatortoken:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Deze operatoroverdracht is bewust begrensd: voldoende om de mobiele
operatorlus en systeemeigen installatie te starten, inclusief `operator.talk.secrets` voor het lezen van de Talk-
configuratie, maar zonder scopes voor koppelingsmutaties en zonder `operator.admin`. Ruimere
toegang voor koppeling/beheer vereist een afzonderlijk goedgekeurd koppelings- of tokenproces. Sla
`hello-ok.auth.deviceTokens` alleen permanent op wanneer de bootstrapauthenticatie via een vertrouwd
transport is uitgevoerd (`wss://` of loopback/lokale koppeling).

Vertrouwde backendclients binnen hetzelfde proces (`client.id: "gateway-client"`,
`client.mode: "backend"`) mogen `device` weglaten bij directe loopbackverbindingen wanneer
ze authenticeren met het gedeelde Gateway-token/wachtwoord. Dit pad is voorbehouden
aan interne RPC's van het besturingsvlak (bijv. sessie-updates van subagents) en voorkomt dat
verouderde basisinstellingen voor CLI-/apparaatkoppeling lokaal backendwerk blokkeren. Externe
clients, clients met browseroorsprong, nodes en expliciete clients met apparaattokens/apparaatidentiteit doorlopen nog steeds
de normale controles voor koppeling en scope-upgrades.

### Workerrol en gesloten protocol

Cloudworkers gebruiken een speciale loopback-ingang via de door de Gateway beheerde,
met een hostsleutel vastgezette SSH-tunnel. Deze accepteert alleen een workeridentiteit en stuurt nooit
algemene authenticatie, Node-gebeurtenissen, operator-RPC's of Plugin-methoden door. Een strikte `connect`
verifieert een gehashte, permanent opgeslagen, kortstondige credential die is gebonden aan de omgeving, de bundle-
hash, de ownerepoch, de RPC-setversie, de vervaldatum en één nullable sessie; afzonderlijk worden
de huidige versie en functieset gecontroleerd. Bij succes wordt een minimale
`worker-hello-ok` geretourneerd; functieonderhandeling staat los van de algemene protocolversie.
Frames blijven kleiner dan 64 KiB, behalve dat een overeengekomen `worker.inference.start`-
frame maximaal 25 MiB mag zijn. De gesloten allowlist bevat `worker.heartbeat`,
`worker.transcript.commit`, `worker.live-event`, `worker.inference.start` en
`worker.inference.cancel`.

Transcriptcommits gebruiken fencing op basis van de ownerepoch, een door de Gateway beheerde sessiekoppeling,
compare-and-swap van het basisblad en duurzame herhaling van sequenties; de Gateway genereert
transcriptitem- en bovenliggende ID's via de normale sessieschrijver. Eigendom en
verval worden bij elke RPC opnieuw gecontroleerd.

### Clientmogelijkheden

Operatorclients kunnen optionele mogelijkheden adverteren in `connect.params.caps`:

- `tool-events`: accepteert gestructureerde gebeurtenissen uit de levenscyclus van tools.
- `inline-widgets`: kan gehoste inline widgetresultaten van tools weergeven.

Clientmogelijkheden beschrijven de verbonden client, niet de autorisatie. Agenttools kunnen vereiste mogelijkheden declareren; de Gateway laat die tools weg tenzij elke vereiste voorkomt in `caps` van de oorspronkelijke client. Runs die vanuit een kanaal afkomstig zijn, hebben geen Gateway-clientmogelijkheden, waardoor tools waarvoor mogelijkheden vereist zijn niet beschikbaar zijn, zelfs wanneer het toolbeleid ze expliciet toestaat.

### Voorbeeld van een Node-verbinding

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Nodes declareren claims voor mogelijkheden wanneer ze verbinding maken:

- `caps`: categorieën op hoog niveau, zoals `camera`, `canvas`, `screen`,
  `location`, `voice`, `talk`.
- `commands`: allowlist met opdrachten voor aanroepen.
- `permissions`: fijnmazige schakelaars (bijv. `screen.record`, `camera.capture`).

De Gateway behandelt deze als claims en dwingt allowlists aan de serverzijde af.

## Rollen en scopes

Zie [Operatorscopes](/nl/gateway/operator-scopes) voor het volledige model voor operatorscopes, controles tijdens goedkeuring en
de semantiek van gedeelde geheimen.

Rollen:

- `operator`: client van het besturingsvlak (CLI/UI/automatisering).
- `node`: host voor mogelijkheden (camera/scherm/canvas/system.run).
- `worker`: cloudhost voor uitvoering via het speciale, gesloten workerprotocol.

Operatorscopes (`src/gateway/operator-scopes.ts`), de volledige gesloten set:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` met `includeSecrets: true` vereist `operator.talk.secrets` (of
`operator.admin`). Wanneer geheimen zijn opgenomen, lees je de credential van de actieve Talk-provider
uit `talk.resolved.config.apiKey`; `talk.providers.<id>.apiKey`
behoudt de bronvorm en kan een SecretRef-object of een geredigeerde tekenreeks zijn.

Door Plugins geregistreerde Gateway-RPC-methoden kunnen hun eigen operatorscope vereisen,
maar deze gereserveerde kernprefixen worden altijd omgezet naar `operator.admin`
(`src/shared/gateway-method-policy.ts`): `config.*`, `exec.approvals.*`,
`wizard.*`, `update.*`.

De methodescope is slechts de eerste controle. Sommige slash-opdrachten die via
`chat.send` worden bereikt, passen strengere controles op opdrachtniveau toe: permanente schrijfbewerkingen naar `/config set` en
`/config unset` vereisen `operator.admin`, zelfs voor Gateway-clients die
al een lagere operatorscope hebben.

`node.pair.approve` heeft naast de basismethodescope
(`operator.pairing`) een extra scopecontrole tijdens de goedkeuring, gebaseerd op de gedeclareerde
`commands` (`src/infra/node-pairing-authz.ts`) van het openstaande verzoek:

| Gedeclareerde opdrachten                                                                                                      | Vereiste scopes                        |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| geen                                                                                                                          | `operator.pairing`                    |
| gewone opdrachten                                                                                                             | `operator.pairing` + `operator.write` |
| bevat `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` of `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

### Capaciteiten/opdrachten/machtigingen (node)

Nodes declareren claims over capaciteiten wanneer ze verbinding maken:

- `caps`: categorieën van capaciteiten op hoog niveau, zoals `camera`, `canvas`, `screen`,
  `location`, `voice` en `talk`.
- `commands`: lijst met toegestane opdrachten voor aanroepen.
- `permissions`: gedetailleerde schakelaars (bijv. `screen.record`, `camera.capture`).

De Gateway behandelt deze als **claims** en dwingt aan de serverzijde lijsten met toegestane waarden af.
Verbonden nodes kunnen na een geslaagde verbinding of
herverbinding optionele, voor agents zichtbare Plugin- of MCP-tooldescriptors publiceren met `node.pluginTools.update`. Headless nodehosts worden opnieuw gestart om wijzigingen
in de declaratieve MCP-inventaris toe te passen. Deze updatemethode is het enige publicatiepad; descriptors van Plugin-tools worden niet geaccepteerd in
de parameters van `connect`. Elke descriptor moet een providerveilige tool-`name` gebruiken en
een `command` opgeven uit de huidige lijst met toegestane opdrachten van de node. De Gateway vertrouwt de metagegevens van descriptors
van de gekoppelde node, filtert descriptors buiten het goedgekeurde opdrachtoppervlak,
verwijdert ze wanneer de verbinding met de node wordt verbroken en weigert pogingen van operators
om de catalogus van een andere node te wijzigen. Stel `gateway.nodes.pluginTools.enabled: false`
in om door nodes gepubliceerde descriptors te negeren.

Verbonden nodehosts publiceren hun volledige vervangingscatalogus voor Skills met
`node.skills.update`. Deze methode voor de noderol is het enige publicatiepad voor Skills
van nodes; Skills worden niet geaccepteerd in de parameters van `connect`. Elke descriptor bevat een
veilige naam, beschrijving en begrensde `SKILL.md`-inhoud. De Gateway parseert die
inhoud met de normale Skills-loader, neemt deze op in snapshots van Skills voor agents
zolang de node verbonden is en verwijdert deze wanneer de verbinding wordt verbroken. Stel
`gateway.nodes.allowSkills: false` in om door nodes gepubliceerde Skills te negeren.

## Aanwezigheid

- `system-presence` retourneert vermeldingen met de apparaatidentiteit als sleutel, waaronder
  `deviceId`, `roles` en `scopes`, zodat UI's één rij per apparaat kunnen tonen, zelfs
  wanneer het zowel als operator als node verbinding maakt.
- `node.list` bevat optioneel `lastSeenAtMs` en `lastSeenReason`. Verbonden
  nodes rapporteren de huidige verbindingstijd met reden `connect`; gekoppelde nodes kunnen
  ook duurzame achtergrondaanwezigheid rapporteren via een vertrouwde nodegebeurtenis.

Native macOS-nodes kunnen ook geverifieerde `node.presence.activity`-gebeurtenissen verzenden
met een begrensde inactieve invoertijd. De Gateway leidt activiteitstijdstempels af met zijn
eigen klok, maakt de meest recent actieve verbonden Mac beschikbaar via `node.list` en
`node.describe` en zendt `node.presence`-updates uit naar clients met leesscope.
De app verzendt `{ "action": "clear" }` wanneer de gebruiker het delen van activiteit uitschakelt;
de Gateway wist tijdstempels alleen voor precies die geverifieerde nodeverbinding.
Gateways van vóór deze bevestigde actie retourneren deze als niet-afgehandeld, waarna de Mac-
node eenmaal opnieuw verbinding maakt en de opschoning bij het verbreken van de verbinding de oude verbindingsstatus laat verwijderen.
Zie [Aanwezigheid van actieve computer](/nl/nodes/presence) voor selectie, privacy, modelcontext
en gedrag voor het routeren van meldingen.

### Gebeurtenis dat een node op de achtergrond actief was

Nodes roepen `node.event` aan met `event: "node.presence.alive"` om vast te leggen dat een
gekoppelde node tijdens een achtergrondactivatie actief was, zonder deze als verbonden te markeren:

```json
{
  "event": "node.presence.alive",
  "payloadJSON": "{\"trigger\":\"silent_push\",\"sentAtMs\":1737264000000,\"displayName\":\"Peter's iPhone\",\"version\":\"2026.4.28\",\"platform\":\"iOS 18.4.0\",\"deviceFamily\":\"iPhone\",\"modelIdentifier\":\"iPhone17,1\",\"pushTransport\":\"relay\"}"
}
```

`trigger` is een gesloten enumeratie: `background`, `silent_push`, `bg_app_refresh`,
`significant_location`, `manual`, `connect`. Onbekende waarden worden genormaliseerd naar
`background` (`src/shared/node-presence.ts`). De gebeurtenis wordt alleen opgeslagen voor
geverifieerde apparaatsessies van nodes; sessies zonder apparaat of niet-gekoppelde sessies retourneren
`handled: false`.

Geslaagde Gateways retourneren een gestructureerd resultaat:

```json
{
  "ok": true,
  "event": "node.presence.alive",
  "handled": true,
  "reason": "persisted"
}
```

Oudere Gateways retourneren mogelijk alleen `{ "ok": true }` voor `node.event`; behandel dit
als een bevestigde RPC, niet als duurzame opslag van aanwezigheid.

## Scoping van broadcastgebeurtenissen

Door de server gepushte broadcastgebeurtenissen worden op basis van scope afgeschermd, zodat sessies
met alleen een koppelingsscope of alleen een noderol niet passief sessie-inhoud ontvangen
(`src/gateway/server-broadcast.ts`):

- Frames voor chats, agents en toolresultaten (gestreamde `agent`-gebeurtenissen,
  toolresultaatgebeurtenissen) vereisen ten minste `operator.read`. Sessies zonder deze scope slaan deze
  frames volledig over.
- Door Plugins gedefinieerde `plugin.*`-broadcasts worden standaard beperkt tot `operator.write` of
  `operator.admin`; expliciete vermeldingen zoals
  `plugin.approval.requested` / `plugin.approval.resolved` gebruiken in plaats daarvan
  `operator.approvals`.
- Status-/transportgebeurtenissen (`heartbeat`, `presence`, `tick`, de levenscyclus
  van verbinden/verbreken) blijven onbeperkt, zodat de transportstatus voor elke
  geverifieerde sessie waarneembaar is.
- Onbekende families van broadcastgebeurtenissen worden standaard op basis van scope afgeschermd (fail-closed),
  tenzij een geregistreerde handler deze beperking expliciet versoepelt.

Elke clientverbinding behoudt een eigen volgnummer per client, zodat broadcasts
op die socket monotoon geordend blijven, zelfs wanneer verschillende clients
verschillende, op scope gefilterde subsets van de gebeurtenisstroom zien.

## Families van RPC-methoden

`hello-ok.features.methods` is een conservatieve ontdekkingslijst die is opgebouwd uit
`src/gateway/server-methods-list.ts` plus geëxporteerde methoden van geladen Plugins/kanalen
— het is geen gegenereerde dump van elke methode, en sommige methoden (bijvoorbeeld
`push.test`, `web.login.start`, `web.login.wait`, `sessions.usage`)
zijn bewust uitgesloten van ontdekking, hoewel het echte, aanroepbare
methoden zijn. Behandel dit als functieontdekking, niet als een volledige opsomming van
`src/gateway/server-methods/*.ts`.

<AccordionGroup>
  <Accordion title="Systeem en identiteit">
    - `health` retourneert de gecachte of onlangs gepeilde momentopname van de Gateway-status.
    - `diagnostics.stability` retourneert de recente, begrensde recorder voor diagnostische stabiliteit: gebeurtenisnamen, aantallen, bytegroottes, geheugenmetingen, wachtrij-/sessiestatus, kanaal-/Pluginnamen, sessie-id's. Geen chattekst, Webhook-bodies, tooluitvoer, onbewerkte request-/response-bodies, tokens, cookies of geheimen. Vereist `operator.read`.
    - `status` retourneert het Gateway-overzicht in `/status`-stijl; gevoelige velden alleen voor operatorclients met beheerdersscope.
    - `gateway.identity.get` retourneert de apparaatidentiteit van de Gateway die wordt gebruikt door relay- en koppelingsflows.
    - `system-presence` retourneert de huidige momentopname van aanwezigheid voor verbonden operator-/nodeapparaten.
    - `system-event` voegt een systeemgebeurtenis toe en kan aanwezigheidscontext bijwerken/uitzenden.
    - `last-heartbeat` retourneert de laatst opgeslagen Heartbeat-gebeurtenis.
    - `set-heartbeats` schakelt Heartbeat-verwerking op de Gateway in of uit.
    - `gateway.suspend.prepare` maakt alleen een korte lease voor coöperatieve opschorting wanneer bijgehouden Gateway-werk inactief is. `gateway.suspend.status` controleert die lease en `gateway.suspend.resume` geeft deze vrij na hervatting of een afgebroken hostbewerking.

  </Accordion>

  <Accordion title="Modellen en gebruik">
    - `models.list` retourneert de tijdens runtime toegestane modelcatalogus. Zie de onderstaande weergaven voor "`models.list`".
    - `usage.status` retourneert gebruiksvensters/overzichten van resterende quota van providers.
    - `usage.cost` retourneert geaggregeerde overzichten van kostengebruik voor een datumbereik. Geef `agentId` door voor één agent, of `agentScope: "all"` om geconfigureerde agents te aggregeren.
    - `doctor.memory.status` retourneert de gereedheidsstatus van vectorgeheugen/gecachete embeddings voor de actieve standaardwerkruimte van de agent. Geef `{ "probe": true }` of `{ "deep": true }` alleen door voor een expliciete live-ping naar een embeddingprovider. Geef `{ "agentId": "agent-id" }` door om statistieken van de Dreaming-opslag te beperken tot één agentwerkruimte; bij weglaten worden geconfigureerde Dreaming-werkruimten geaggregeerd.
    - `doctor.memory.dreamDiary`, `doctor.memory.backfillDreamDiary`, `doctor.memory.resetDreamDiary`, `doctor.memory.resetGroundedShortTerm`, `doctor.memory.repairDreamingArtifacts` en `doctor.memory.dedupeDreamDiary` accepteren optioneel `{ "agentId": "agent-id" }`; bij weglaten werken ze op de geconfigureerde standaardwerkruimte van de agent.
    - `doctor.memory.remHarness` retourneert een begrensde, alleen-lezen preview van de REM-harnas voor externe control-plane-clients, inclusief werkruimtepaden, geheugenfragmenten, gerenderde onderbouwde Markdown en kandidaten voor diepgaande promotie. Vereist `operator.read`.
    - `sessions.usage` retourneert gebruiksoverzichten per sessie. Geef `agentId` door voor één agent, of `agentScope: "all"` om geconfigureerde agents samen weer te geven.
      Beide gebruiksmethoden accepteren `mode: "specific"` met een IANA-`timeZone` voor kalenderdaggrenzen en buckets die rekening houden met zomertijd. `utcOffset` blijft ondersteund voor oudere clients en als terugval wanneer de Gateway-runtime de aangevraagde zone niet herkent.
    - `sessions.usage.timeseries` retourneert tijdreeksgebruik voor één sessie.
    - `sessions.usage.logs` retourneert gebruikslogvermeldingen voor één sessie.

  </Accordion>

  <Accordion title="Kanalen en aanmeldhelpers">
    - `channels.status` retourneert ingebouwde + gebundelde statusoverzichten van kanalen/Plugins.
    - `channels.logout` meldt een specifiek kanaal/account af wanneer het kanaal dit ondersteunt.
    - `web.login.start` start een QR-/webaanmeldflow voor de huidige webkanaalprovider met QR-ondersteuning.
    - `web.login.wait` wacht totdat die flow is voltooid en start bij succes het kanaal.
    - `push.test` verzendt een test-APNs-push naar een geregistreerde iOS-node.
    - `voicewake.get` retourneert de opgeslagen activeringswoordtriggers.
    - `voicewake.set` werkt activeringswoordtriggers bij en zendt de wijziging uit.

  </Accordion>

  <Accordion title="Pluginbeheer">
    - `plugins.list` (`operator.read`) retourneert de inventaris van geïnstalleerde plugins, plus lokaal samengestelde officiële aanbevelingen, diagnostische gegevens en of de huidige installatiemodus wijzigingen toestaat.
    - `plugins.search` (`operator.read`) zoekt naar installeerbare families van ClawHub-codeplugins en -bundelplugins. Geef een niet-lege `query` en een optionele `limit` van 1 tot 100 door.
    - `plugins.install` (`operator.admin`) installeert een officiële catalogusvermelding met `{ source: "official", pluginId }` of een ClawHub-pakket met `{ source: "clawhub", packageName, version?, acknowledgeClawHubRisk? }`. Bij ClawHub-installaties blijven de Gateway-controles voor vertrouwen, integriteit en installatiebeleid behouden. Na een geslaagde installatie moet de Gateway opnieuw worden gestart.
    - `plugins.setEnabled` (`operator.admin`) wijzigt met `{ pluginId, enabled }` het inschakelbeleid van één geïnstalleerde plugin. Het antwoord bevat de bijgewerkte catalogusvermelding, metadata voor het opnieuw starten en eventuele waarschuwingen over de sleufselectie.
    - `plugins.uninstall` (`operator.admin`) verwijdert met `{ pluginId }` één extern geïnstalleerde plugin: configuratieverwijzingen, de installatierecord en beheerde bestanden. Gebundelde plugins kunnen niet worden verwijderd, maar alleen worden uitgeschakeld. Het antwoord vermeldt de verwijderingsacties en vereist altijd dat de Gateway opnieuw wordt gestart.

  </Accordion>

  <Accordion title="Berichten en logboeken">
    - `send` is de RPC voor rechtstreekse uitgaande aflevering voor verzendingen buiten de chatrunner die op een kanaal, account en thread zijn gericht.
    - `logs.tail` retourneert het geconfigureerde uiteinde van het Gateway-bestandslogboek, met besturingselementen voor cursor/limiet en het maximale aantal bytes.

  </Accordion>

  <Accordion title="Operatorterminal">
    - `terminal.open` start een host-PTY voor een expliciete `agentId` of de standaardagent en retourneert de bepaalde agent, werkmap, shell en isolatiestatus.
    - `terminal.input`, `terminal.resize` en `terminal.close` werken alleen op sessies die eigendom zijn van de aanroepende verbinding.
    - `terminal.upload` accepteert één base64-bestand van maximaal 16 MiB, plaatst dit in een persoonlijke tijdelijke map met een levensduur van 24 uur op de Gateway van de sessie of de host van de gekoppelde Node, en retourneert het absolute pad. De aanroeper moet dat pad nog steeds plakken of anderszins gebruiken; de RPC schrijft nooit terminalinvoer en voert geen opdracht uit.
    - `terminal.data`- en `terminal.exit`-gebeurtenissen worden alleen gestreamd naar de verbinding die eigenaar is van de sessie.
    - Sessies waarvan de verbinding wordt verbroken, worden losgekoppeld en niet beëindigd: ze kunnen gedurende `gateway.terminal.detachedSessionTimeoutSeconds` opnieuw worden gekoppeld (standaard 300; `0` herstelt beëindiging bij verbreking van de verbinding), terwijl recente uitvoer zich ophoopt in een begrensde buffer aan de serverzijde.
    - `terminal.list` retourneert koppelbare sessies; `terminal.attach` koppelt een actieve of losgekoppelde sessie opnieuw aan de aanroepende verbinding en retourneert de herhalingsbuffer (overname in tmux-stijl — een vorige actieve eigenaar ontvangt `terminal.exit` met reden `detached`); `terminal.text` leest de buffer als platte tekst zonder de sessie te koppelen.
    - Elke terminalmethode vereist `operator.admin`; `gateway.terminal.enabled` moet expliciet waar zijn. Volledig gesandboxte agents worden geweigerd en een wijziging van het agentbeleid sluit bestaande en lopende PTY's, inclusief losgekoppelde PTY's.

  </Accordion>

  <Accordion title="Spraak en TTS">
    - `talk.catalog` retourneert de alleen-lezen catalogus met Talk-providers voor spraak, streamingtranscriptie en realtime spraak: canonieke provider-id's, registeraliassen, labels, configuratiestatus, een optioneel `ready`-resultaat op groepsniveau, beschikbare model-/spraak-id's, canonieke modi, transporten, denkstrategieën en vlaggen voor realtime audio/mogelijkheden, zonder providergeheimen te retourneren of de globale configuratie te wijzigen. Huidige Gateways stellen `ready` in nadat de runtimeproviderselectie is toegepast; beschouw het ontbreken ervan als niet-geverifieerd op oudere Gateways.
    - `talk.config` retourneert de effectieve payload van de Talk-configuratie; `includeSecrets` vereist `operator.talk.secrets` (of `operator.admin`).
    - `talk.session.create` maakt een Talk-sessie in beheer van de Gateway voor `realtime/gateway-relay`, `transcription/gateway-relay` of `stt-tts/managed-room`. Voor `stt-tts/managed-room` moeten `operator.write`-aanroepers die `sessionKey` doorgeven, ook `spawnedBy` doorgeven voor zichtbaarheid van de sessiesleutel binnen het bereik; het maken van `sessionKey` zonder bereik en `brain: "direct-tools"` vereisen `operator.admin`.
    - `talk.session.join` valideert een sessietoken voor een beheerde ruimte, verzendt indien nodig `session.ready` of `session.replaced`, en retourneert metadata over de ruimte en sessie plus recente Talk-gebeurtenissen, maar nooit het token in platte tekst of de hash ervan.
    - `talk.session.appendAudio` voegt base64-PCM-invoeraudio toe aan realtime relay- en transcriptiesessies in beheer van de Gateway.
    - `talk.session.startTurn`, `talk.session.endTurn` en `talk.session.cancelTurn` sturen de levenscyclus van beurten in beheerde ruimten aan, waarbij verouderde beurten worden geweigerd voordat de status wordt gewist.
    - `talk.session.cancelOutput` stopt de audio-uitvoer van de assistent, voornamelijk voor door VAD aangestuurde onderbrekingen in Gateway-relaysessies.
    - `talk.session.submitToolResult` voltooit een provider-toolaanroep die door een realtime relaysessie in beheer van de Gateway is verzonden. De aanvraag wacht op elk asynchroon voltooiingssignaal dat door de providerbridge beschikbaar wordt gesteld; mislukte inzendingen houden de gekoppelde uitvoering actief en verzenden geen gebeurtenis voor een geslaagd toolresultaat. Geef `options: { willContinue: true }` door voor tussentijdse tooluitvoer of `options: { suppressResponse: true }` wanneer de providerbridge ondersteuning voor onderdrukking aangeeft en het resultaat geen nieuw antwoord mag starten.
    - `talk.session.steer` stuurt spraakbesturing voor een actieve uitvoering naar een door een agent ondersteunde Talk-sessie in beheer van de Gateway: `{ sessionId, text, mode? }`, waarbij `mode` gelijk is aan `status`, `steer`, `cancel` of `followup`; als de modus wordt weggelaten, wordt deze geclassificeerd op basis van de gesproken tekst.
    - `talk.session.close` sluit een relay-, transcriptie- of beheerde-ruimtesessie in beheer van de Gateway en verzendt afsluitende Talk-gebeurtenissen.
    - `talk.mode` stelt de huidige status van de Talk-modus in en zendt deze uit voor WebChat-/Control UI-clients.
    - `talk.client.create` maakt of hervat een realtime providersessie in beheer van de client met `webrtc` of `provider-websocket`, terwijl de Gateway de referenties, instructies, het toolbeleid en de geretourneerde `voiceSessionId` beheert. Clients geven `sessionKey` door en hergebruiken `voiceSessionId` wanneer ze tijdens één aanroep het providertransport vervangen.
    - `talk.client.transcript` voegt één afgerond `{ role, text }`-item toe aan de normale agentsessie. De vereiste `entryId` is idempotent binnen `voiceSessionId`; nieuwe pogingen dupliceren geen transcriptberichten.
    - `talk.client.close` sluit de logische spraaksessie nadat openstaande transcriptschrijfbewerkingen zijn voltooid. Sluiten is idempotent en kan een alleen-wijzigingssamenvatting van de aanroep afleveren bij het laatste niet-WebChat-kanaal van de sessie.
    - `talk.client.toolCall` laat realtime transporten in beheer van de client provider-toolaanroepen doorsturen naar het Gateway-beleid. De eerste ondersteunde tool is `openclaw_agent_consult`; clients ontvangen een uitvoerings-id en wachten op normale gebeurtenissen in de chatlevenscyclus voordat ze het providerspecifieke toolresultaat indienen. Spraakgebonden acties met grote impact retourneren `VOICE_CONFIRMATION_REQUIRED:<id>` totdat een latere afgeronde gebruikersuiting die exacte actie expliciet bevestigt en de volgende raadpleging de `confirmationId` aanlevert.
    - `talk.client.steer` stuurt spraakbesturing voor een actieve uitvoering voor realtime transporten in beheer van de client. De Gateway bepaalt de actieve ingesloten uitvoering op basis van `sessionKey` en retourneert een gestructureerd geaccepteerd/geweigerd resultaat in plaats van bijsturing stilzwijgend te negeren.
    - `talk.event` is het enige Talk-gebeurteniskanaal voor realtime, transcriptie, STT/TTS, beheerde ruimten, telefonie en vergaderadapters.
    - `talk.speak` synthetiseert spraak via de actieve Talk-spraakprovider.
    - `tts.status` retourneert de ingeschakelde status van TTS, de actieve provider, fallbackproviders en de configuratiestatus van providers.
    - `tts.providers` retourneert de zichtbare inventaris van TTS-providers.
    - `tts.enable` en `tts.disable` schakelen de status van TTS-voorkeuren om.
    - `tts.setProvider` werkt de voorkeursprovider voor TTS bij.
    - `tts.convert` voert een eenmalige conversie van tekst naar spraak uit.
    - `tts.speak` (`operator.write`) zet een niet-lege `text` om met de geconfigureerde algemene TTS-providerketen en retourneert één volledige clip inline als `audioBase64`, plus `provider` en optionele metadata voor `outputFormat`, `mimeType` en `fileExtension`. In tegenstelling tot `tts.convert` retourneert deze geen lokaal Gateway-pad; in tegenstelling tot `talk.speak` vereist deze geen Talk-provider. Tekst langer dan `tts.maxTextLength` retourneert `INVALID_REQUEST`; synthesefouten retourneren `UNAVAILABLE`.

  </Accordion>

  <Accordion title="Secrets, configuratie, updates en wizard">
    - `secrets.reload` lost actieve SecretRefs opnieuw op en publiceert atomair runtime-status die rekening houdt met de eigenaar. In aanmerking komende fouten van eigenaren kunnen met `warningCount` worden gepubliceerd als koude of verouderde degradatie; strikte of niet-toegewezen fouten wijzen het opnieuw laden af en behouden de actieve momentopname.
    - `secrets.resolve` lost geheime toewijzingen voor opdrachtdoelen op voor een specifieke set opdrachten/doelen.
    - `config.get` retourneert de huidige configuratiemomentopname op schijf, het onbewerkte `hash` van het hoofdbestand, de opgeloste `configRevisionHash` en de optionele `appliedConfigHash` voor de opgeloste revisie die door de actieve Gateway-runtime is geaccepteerd.
    - `config.set` schrijft een gevalideerde configuratiepayload.
    - `config.patch` voegt een gedeeltelijke configuratie-update samen. Voor destructieve vervanging van arrays is het betreffende pad vereist in `replacePaths`; geneste arrays onder array-items gebruiken `[]`-paden zoals `agents.entries.*.skills`.
    - `config.apply` valideert en vervangt de volledige configuratiepayload.
    - `config.schema` retourneert de live payload van het configuratieschema die wordt gebruikt door de Control UI en CLI-hulpmiddelen: schema, `uiHints`, versie, generatiemetadata en, indien laadbaar, schema-metadata van plugins en kanalen. Deze bevat `title`- / `description`-metadata uit dezelfde labels/helptekst als de UI, inclusief vertakkingen voor geneste objecten, jokertekens, array-items en `anyOf` / `oneOf` / `allOf`-composities wanneer bijpassende velddocumentatie bestaat.
    - `config.schema.lookup` retourneert een tot één configuratiepad beperkte opzoekpayload: genormaliseerd pad, een oppervlakkig schemaknooppunt, overeenkomende hint plus `hintPath`, optionele `reloadKind` en samenvattingen van directe onderliggende items voor detailnavigatie in de UI/CLI. `reloadKind` is een van `restart`, `hot` of `none` (`src/config/schema.ts`) en weerspiegelt de planner voor het opnieuw laden van de Gateway-configuratie voor het aangevraagde pad. Schemakooppunten voor opzoekacties behouden de gebruikersgerichte documentatie en algemene validatievelden (`title`, `description`, `type`, `enum`, `const`, `format`, `pattern`, grenzen voor getallen/tekenreeksen/arrays/objecten, `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`). Samenvattingen van onderliggende items tonen `key`, het genormaliseerde `path`, `type`, `required`, `hasChildren`, de optionele `reloadKind`, plus de overeenkomende `hint` / `hintPath`.
    - `update.run` voert de Gateway-updateflow uit en plant alleen een herstart als de update is geslaagd; aanroepers met een sessie kunnen `continuationMessage` opnemen, zodat bij het opstarten één vervolgstap van de agent wordt hervat via de wachtrij voor voortzetting na een herstart. Updates via pakketbeheerders en begeleide updates van git-checkouts vanuit het besturingsvlak gebruiken een losgekoppelde overdracht aan een beheerde service in plaats van de pakketstructuur te vervangen of checkout-/build-uitvoer binnen de actieve Gateway te wijzigen. Een gestarte overdracht retourneert `ok: true` met `result.reason: "managed-service-handoff-started"` en `handoff.status: "started"`. Een tweede gelijktijdige `update.run` die door hetzelfde Gateway-proces wordt afgehandeld, retourneert `ok: false` met `result.reason: "managed-service-handoff-already-running"` en `handoff.status: "already-running"`; de voortzetting ervan wordt niet geaccepteerd, zodat de aanroeper het opnieuw kan proberen nadat de actieve update is voltooid. Zelfstandige CLI-updaters en vervangende Gateway-processen vallen buiten deze proceslokale beveiliging. Niet-beschikbare of mislukte overdrachten retourneren `ok: false` met `managed-service-handoff-unavailable` of `managed-service-handoff-failed`, plus `handoff.command` wanneer een handmatige shell-update vereist is. Niet-beschikbaar betekent dat OpenClaw geen veilige supervisorgrens of duurzame service-identiteit heeft, zoals `OPENCLAW_SYSTEMD_UNIT` voor systemd. Tijdens een gestarte overdracht kan de herstartmarkering kort `stats.reason: "restart-health-pending"` melden; de voortzetting wordt uitgesteld totdat de CLI de opnieuw gestarte Gateway verifieert en de definitieve `ok`-markering schrijft.
    - `update.status` vernieuwt en retourneert de nieuwste markering voor een updateherstart, inclusief de actieve versie na de herstart wanneer die beschikbaar is.
    - `wizard.start`, `wizard.next`, `wizard.status` en `wizard.cancel` ontsluiten de onboardingwizard via WS-RPC.

  </Accordion>

  <Accordion title="Helpers voor agents en werkruimten">
    - `agents.list` retourneert agent-items die zichtbaar zijn voor de Gateway, inclusief effectieve model-/runtime-metadata en optionele semantische `kind` (`agent` of `system`). Clients kondigen de handshake-mogelijkheid `agent-kind` aan om het volledige getypeerde overzicht te ontvangen; clients zonder deze mogelijkheid behouden het verouderde, voor selectors veilige overzicht zonder systeemrijen. Clients die rekening houden met het type sluiten `system`-rijen uit van gewone selectors, maar behouden ze in diagnostische weergaven. Oudere v4-Gateways kunnen rijen zonder `kind` retourneren.
    - `agents.create`, `agents.update` en `agents.delete` beheren agentrecords en de koppeling met werkruimten.
    - `agents.files.list`, `agents.files.get` en `agents.files.set` beheren de bootstrapbestanden van de werkruimte die voor een agent beschikbaar worden gesteld.
    - `audit.activity.list` retourneert het geversioneerde activiteitenregister met alleen metadata; `audit.list` blijft de compatibiliteitsveilige RPC voor uitvoeringen/hulpmiddelen.
    - `agents.workspace.list` en `agents.workspace.get` (`operator.read`) bieden alleen-lezen, gepagineerd bladeren door de werkruimtemap van een agent voor clients in het vertrouwde operatordomein dat wordt beschreven in [Operatorbereiken](/nl/gateway/operator-scopes). Aanvragen accepteren alleen paden relatief aan de werkruimte; leesbewerkingen blijven beperkt tot de via realpath bepaalde hoofdmap van de werkruimte (ontsnappingen via symbolische en harde koppelingen worden geweigerd), hebben een maximale grootte en zijn beperkt tot UTF-8-tekst plus gangbare afbeeldingstypen (base64). Antwoorden maken het hostpad van de werkruimte niet bekend. Deze naamruimte bevat geen schrijfbewerkingen.
    - `tasks.list`, `tasks.get` en `tasks.cancel` stellen het Gateway-takenregister beschikbaar aan SDK- en operatorclients. Zie hieronder [RPC's voor het takenregister](#task-ledger-rpcs).
    - `artifacts.list`, `artifacts.get` en `artifacts.download` bieden uit transcripties afgeleide artefactsamenvattingen en downloads voor een expliciet `sessionKey`-, `runId`- of `taskId`-bereik. Query's voor uitvoeringen en taken bepalen de bijbehorende sessie aan de serverzijde en retourneren alleen transcriptiemedia met overeenkomende herkomst; onveilige of lokale URL-bronnen retourneren niet-ondersteunde downloads in plaats van ze aan de serverzijde op te halen.
    - `environments.list` en `environments.status` behouden de detectie van Gateway-lokale en Node-omgevingen. Geconfigureerde cloudworkers en duurzame records die door eerdere profielen zijn achtergelaten, voegen `worker`-metadata toe met `providerId`, optionele `leaseId`, `state`, `ageMs`, optionele `idleMs` en `attachedSessionIds`. De levenscyclusstatussen van workers zijn `requested`, `provisioning`, `bootstrapping`, `ready`, `attached`, `idle`, `draining`, `destroying`, `destroyed`, `failed` en `orphaned`.
    - `environments.create` (`{ profileId, idempotencyKey }`) richt een worker in vanuit een geconfigureerd providerprofiel van een Plugin; nieuwe pogingen met dezelfde sleutel hergebruiken de duurzame bewerking. `environments.destroy` (`{ environmentId }`) vraagt om idempotente ontmanteling van een duurzame workeromgeving. Beide vereisen `operator.admin`, zijn schrijfbewerkingen van het besturingsvlak en retourneren dezelfde vorm van de omgevingssamenvatting die door statusantwoorden wordt gebruikt.
    - `agent.identity.get` retourneert de effectieve assistentidentiteit voor een agent of sessie.
    - `agent.wait` wacht tot een uitvoering is voltooid en retourneert de eindmomentopname wanneer die beschikbaar is.

  </Accordion>

  <Accordion title="Sessiebeheer">
    - `sessions.list` retourneert de huidige sessie-index, inclusief `agentRuntime`-metadata per rij wanneer een backend voor de agentruntime is geconfigureerd. Wanneer plaatsing op cloudworkers is ingeschakeld of duurzame herstelstatus bestaat, bevatten sessierijen ook een afgesloten `placement`-status (`local`, `requested`, `provisioning`, `syncing`, `starting`, `active`, `draining`, `reconciling`, `reclaimed` of `failed`), plus statusafhankelijke velden voor omgeving, owner-epoch, werkruimte, bundel, ACK-cursor of herstel.
    - `sessions.subscribe` en `sessions.unsubscribe` schakelen abonnementen op sessiewijzigingsgebeurtenissen in of uit voor de huidige WS-client.
    - `sessions.messages.subscribe` en `sessions.messages.unsubscribe` schakelen abonnementen op transcript-/berichtgebeurtenissen in of uit voor één sessie. Geef `includeApprovals: true` door om ook opgeschoonde `session.approval`-levenscyclusgebeurtenissen te ontvangen voor goedkeuringen waarvan het opgeslagen publiek exact die sessie omvat en waarvan de reviewerbinding de abonnerende client autoriseert. Het antwoord op het abonnement bevat dan een begrensde openstaande `approvalReplay`; deze is gezaghebbend wanneer `truncated` false is. De opt-in geldt per abonnementsaanroep en blijft niet behouden: opnieuw abonneren op dezelfde sessie zonder `includeApprovals: true` verwijdert een bestaand goedkeuringsabonnement. Naast de normale autorisatie om de sessie te lezen, vereist deze opt-in `operator.admin`, of `operator.approvals` op een gekoppeld apparaat.
    - `sessions.preview` retourneert begrensde transcriptvoorbeelden voor specifieke sessiesleutels.
    - `sessions.describe` retourneert één Gateway-sessierij voor een exacte sessiesleutel.
    - `sessions.resolve` herleidt of canonicaliseert een sessiedoel.
    - `sessions.create` maakt een nieuwe sessievermelding. Optionele waarden voor `model` en `thinkingLevel` slaan de initiële model- en redeneersoverschrijvingen atomair op. `worktree: true` richt een beheerde worktree in; optionele `worktreeBaseRef`/`worktreeName` selecteren de basisreferentie en branchnaam, en `execNode` (`operator.admin`) bindt sessie-uitvoering aan een Node-host. De gemaakte worktree wordt in het resultaat teruggegeven en in de sessierij opgeslagen (`worktree: { id, branch, repoRoot }`). Wanneer de vermelding is gemaakt maar de geneste initiële `chat.send` wordt geweigerd, bevat het geslaagde resultaat `runStarted: false` en `runError`; clients kunnen de prompt behouden en het opnieuw proberen met de geretourneerde sessiesleutel. Een aanroeper die `parentSessionKey` met `emitCommandHooks: true` doorgeeft, moet ook de levenscyclusafhandeling van een afzonderlijk child declareren: `succeedsParent: true` beëindigt de parent met `session_end`, terwijl `false` de parent actief houdt en alleen de `session_start` van het child uitzendt. Als `succeedsParent` wordt weggelaten, blijft het verouderde parent-rollovergedrag voor bestaande clients behouden. De afhandeling vereist zowel een parentkoppeling als command-hooks; een fork kan zijn parent niet laten slagen. Het gedrag waarbij de hoofdsessie ter plaatse wordt gereset, blijft ongewijzigd omdat er geen afzonderlijk child wordt gemaakt. Nieuwe rijen krijgen een eenmalig schrijfbare herkomstmarkering voor het maken (`createdVia`, `createdActor`, `createdAt`) vanuit de vertrouwde aanmaakinterface; het overnemen van een bestaande sleutel brengt deze markering nooit opnieuw aan. Voor menselijke profielactoren wordt `createdActor.label` vanuit het huidige gebruikersprofiel herleid wanneer de rij wordt geprojecteerd en nooit in de sessievermelding opgeslagen, zodat profielnamen na een naamswijziging niet uiteenlopen. Sessierijen bevatten ook `parentSessionKey` (navigatieparent, opgeslagen), `controlOwnerSessionKey` (runtimecontroller wanneer actief), `forkSource` (exacte bronsleutel + transcriptgeneratie voor forks) en `previousSessionId` (eerdere transcriptgeneratie onder dezelfde sleutel).
    - `sessions.dispatch` (`operator.admin`) verplaatst een bestaande lokale OpenClaw-sessie met een door de sessie beheerde worktree naar een geconfigureerd cloudworkerprofiel. Geef `{ key, profileId, agentId? }` door. De methode ontbreekt wanneer geen workerprofiel is geconfigureerd, sluit lokale toelating van beurten voordat actief werk wordt afgehandeld en retourneert pas nadat de plaatsing `active`-workereigenaarschap heeft bereikt. Dispatch werkt één kant op; terughalen van worker naar lokaal maakt geen deel uit van deze RPC.
    - `sessions.groups.list`, `sessions.groups.put`, `sessions.groups.rename` en `sessions.groups.delete` beheren de aangepaste sessiegroepencatalogus van de Gateway (namen + weergavevolgorde). Het lidmaatschap blijft opgeslagen in het `category`-veld van elke sessie; hernoemen en verwijderen werken lidsessies aan serverzijde bij.
    - `sessions.send` stuurt een bericht naar een bestaande sessie.
    - `sessions.steer` is de variant voor onderbreken en bijsturen van een actieve sessie.
    - `sessions.abort` breekt actief werk voor een sessie af. Geef `key` plus optioneel `runId` door, of alleen `runId` voor actieve uitvoeringen die de Gateway aan een sessie kan koppelen. Als `runId` wordt opgegeven, blijft de annulering beperkt tot die uitvoering. Stel `clearQueued: true` in bij een niet-globaal verzoek met alleen een sleutel om ook vervolg- en lane-wachtrijen van die sessie te verwijderen. Bestaande aanroepers die `clearQueued` weglaten, behouden deze wachtrijen. De letterlijke sleutel `global` behoudt de bestaande agentgekwalificeerde eigendomsregels van `chat.abort` en voert geen niet-globale opschoning van vervolg- of lane-wachtrijen uit.
    - `sessions.patch` werkt sessiemetadata/-overschrijvingen bij en rapporteert het herleide canonieke model plus de effectieve `agentRuntime`. Afstammingsgegevens van spawn-acties (`spawnedBy`, `spawnedWorkspaceDir`, `spawnedCwd`, `spawnDepth`, `subagentRole`, `subagentControlScope`) kunnen niet langer openbaar worden gepatcht; deze gegevens worden eenmaal door vertrouwde aanmaakpaden geschreven en verzoeken die ze nog steeds meesturen, worden geweigerd.
    - `sessions.reset`, `sessions.delete` en `sessions.compact` voeren sessieonderhoud uit.
    - `sessions.get` retourneert de volledige opgeslagen sessierij.
    - Chatuitvoering gebruikt nog steeds `chat.history`, `chat.send`, `chat.abort` en `chat.inject`. `chat.history` wordt voor weergave genormaliseerd voor UI-clients: inline directivetags worden uit zichtbare tekst verwijderd, XML-payloads voor toolaanroepen in platte tekst (`<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` en afgekorte toolaanroepblokken) en gelekte ASCII-/volledige-breedte-modelbesturingstokens worden verwijderd, assistentrijen die uitsluitend een stil token bevatten (exact `NO_REPLY` / `no_reply`) worden weggelaten en te grote rijen kunnen door tijdelijke aanduidingen worden vervangen.
    - `chat.message.get` is de aanvullende begrensde lezer voor volledige berichten van één zichtbare transcriptvermelding. Geef `sessionKey` door, optioneel `agentId` wanneer de sessieselectie agentgebonden is, en een transcript-`messageId` die eerder via `chat.history` beschikbaar is gesteld; de Gateway retourneert dezelfde voor weergave genormaliseerde projectie zonder de afkappingslimiet voor lichtgewicht geschiedenis wanneer de opgeslagen vermelding nog beschikbaar en niet te groot is.
    - `chat.toolTitles` retourneert korte doeltitels voor toolaanroepen die in de Control UI worden weergegeven (in batches, maximaal 24 items met begrensde invoer). De functie is opt-in via `gateway.controlUi.toolTitles` (standaard uitgeschakeld); uitgeschakelde Gateways beantwoorden `{ titles: {}, disabled: true }` zonder modelaanroep, zodat clients niet meer blijven vragen. Wanneer dit is ingeschakeld, gebruiken titels de standaardroutering voor utility-modellen: een expliciet geconfigureerde `utilityModel` (een beslissing van de operator die, net als alle utility-taken, begrensde taakinhoud naar de gekozen provider kan sturen), anders de gedeclareerde standaard voor kleine modellen van de sessieprovider, zodat niet impliciet een nieuwe uitgaande bestemming ontstaat; een lege `utilityModel` schakelt ze volledig uit. Titels vallen nooit terug op het primaire model. Resultaten worden gecachet in de statusdatabase per agent, met toolnaam + invoer als sleutel, zodat herhaalde weergaven dezelfde aanroepen nooit opnieuw in rekening brengen.
    - `chat.send` accepteert een eenmalige `fastMode: "auto"` om de snelle modus te gebruiken voor modelaanroepen die vóór de automatische grens worden gestart, en start latere nieuwe pogingen, fallbacks, toolresultaat- of vervolgaanroepen vervolgens zonder snelle modus. De grens is standaard 60 seconden (`DEFAULT_FAST_MODE_AUTO_ON_SECONDS`) en kan per model worden geconfigureerd met `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds`. Een `chat.send`-aanroeper kan een eenmalige `fastAutoOnSeconds` doorgeven om de grens voor dat verzoek te overschrijven. Geef `queueMode` (`steer`, `followup`, `collect` of `interrupt`) door om de opgeslagen wachtrijmodus alleen voor dit verzoek te overschrijven; expliciete bijstuuracties in de Control UI gebruiken `queueMode: "steer"`. Interactieve clients kunnen `expectedLeafEntryId` doorgeven met het blad van de actieve transcriptbranch dat ze weergeven, of `null` voor een gezaghebbend leeg transcript; de Gateway weigert de verzending met `details.reason: "active-leaf-changed"` als een andere client eerst van branch is gewisseld.

  </Accordion>

  <Accordion title="Apparaatkoppeling en apparaattokens">
    - `device.pair.list` retourneert wachtende en goedgekeurde gekoppelde apparaten.
    - `device.pair.setupCode` maakt een mobiele installatiecode en standaard een PNG-QR-data-URL. Hiervoor is `operator.admin` vereist en de methode wordt opzettelijk weggelaten uit de geadverteerde ontdekking. Het resultaat bevat `setupCode`, optioneel `qrDataUrl`, `gatewayUrl`, het niet-geheime `auth`-label en `urlSource`.
    - `device.pair.approve`, `device.pair.reject` en `device.pair.remove` beheren records voor apparaatkoppeling.
    - `device.pair.rename` wijst een operatorlabel (`{ deviceId, label }`) toe dat de voorkeur krijgt boven de door de client gerapporteerde weergavenaam en behouden blijft na apparaatherstel of hernieuwde goedkeuring.
    - `device.token.rotate` roteert een token van een gekoppeld apparaat binnen de grenzen van de goedgekeurde rol en het bereik van de aanroeper.
    - `device.token.revoke` trekt een token van een gekoppeld apparaat in binnen de grenzen van de goedgekeurde rol en het bereik van de aanroeper.

    De installatiecode bevat een kortlevende bootstrapreferentie. Clients mogen deze niet
    loggen of na het koppelingsproces bewaren.

  </Accordion>

  <Accordion title="Node-koppeling, aanroepen en wachtend werk">
    - `node.pair.list`, `node.pair.approve`, `node.pair.reject` en `node.pair.remove` behandelen goedkeuringen van Node-capaciteiten. `node.pair.request` en `node.pair.verify` zijn in 2026.7 verwijderd, samen met de zelfstandige opslag voor Node-koppelingen; wachtende verzoeken worden door de Gateway gemaakt wanneer Nodes verbinding maken.
    - `node.list` en `node.describe` retourneren de bekende/verbonden Node-status.
    - `node.rename` werkt het label van een gekoppelde Node bij.
    - `node.invoke` stuurt een opdracht door naar een verbonden Node.
    - `node.invoke.result` retourneert het resultaat van een aanroepverzoek.
    - `mcp.tools.call.v1` is de headless Node-hostopdracht voor het aanroepen van een geconfigureerde Node-lokale MCP-tool. Deze wordt via `node.invoke` doorgegeven, vereist dat de Node de opdracht declareert en blijft onderworpen aan koppelingsgoedkeuring en `gateway.nodes.commands.deny`.
    - `node.event` stuurt gebeurtenissen die van een Node afkomstig zijn terug naar de Gateway.
    - `node.pluginTools.update` is het enige publicatiepad voor het vervangen van de agentzichtbare Plugin-/MCP-tooldescriptors van de verbonden Node; `connect`-parameters bevatten deze niet.
    - `node.pending.pull` en `node.pending.ack` zijn de wachtrij-API's voor verbonden Nodes.
    - `node.pending.enqueue` en `node.pending.drain` beheren duurzaam wachtend werk voor offline/niet-verbonden Nodes.

  </Accordion>

  <Accordion title="Goedkeuringscategorieën">
    - `approval.history` retourneert de nieuwste definitieve goedkeuringen eerst, die 30 dagen worden bewaard voor uitvoerings-, plugin- en systeemagentaanvragen (bereik `operator.approvals`). Het ondersteunt cursorpaginering en een optioneel typefilter; openstaande goedkeuringen zijn geen geschiedenisrijen.
    - `approval.get` en `approval.resolve` zijn de type-onafhankelijke duurzame goedkeuringsmethoden (bereik `operator.approvals`). `approval.get` retourneert een opgeschoonde projectie van een openstaande of bewaarde definitieve goedkeuring met een stabiele `urlPath`; `approval.resolve` accepteert de canonieke goedkeurings-id, een expliciete `kind` en een beslissing, past afhandeling toe waarbij het eerste antwoord geldt en retourneert altijd het vastgelegde canonieke resultaat.
    - `exec.approval.request`, `exec.approval.get`, `exec.approval.list` en `exec.approval.resolve` omvatten eenmalige uitvoeringsgoedkeuringsaanvragen en het opzoeken/opnieuw afspelen van openstaande goedkeuringen. Het zijn adapters aan de protocolgrens boven op hetzelfde duurzame goedkeuringsregister.
    - `exec.approval.waitDecision` wacht op één openstaande uitvoeringsgoedkeuring en retourneert de definitieve beslissing (of `null` bij een time-out).
    - `exec.approvals.get` en `exec.approvals.set` beheren momentopnamen van het Gateway-beleid voor uitvoeringsgoedkeuringen.
    - `exec.approvals.node.get` en `exec.approvals.node.set` beheren het node-lokale beleid voor uitvoeringsgoedkeuringen via node-relayopdrachten.
    - `plugin.approval.request`, `plugin.approval.list`, `plugin.approval.waitDecision` en `plugin.approval.resolve` omvatten door plugins gedefinieerde goedkeuringsflows.

  </Accordion>

  <Accordion title="Opdrachten voor de Control UI">
    - `ui.command` laat een `operator.write`-aanroeper getypeerde lay-out- en navigatieopdrachten verzenden naar verbonden Control UI-clients die de mogelijkheid `ui-commands` kenbaar maken.
    - Opdrachten omvatten het splitsen, sluiten en focussen van deelvensters, de zichtbaarheid van de zijbalk, de zichtbaarheid en dokpositie van terminal- en browserpanelen en sessienavigatie.
    - Protocol v1 stuurt de opdracht bewust door naar elke verbonden geschikte Control UI. Als er geen is verbonden, mislukt de aanvraag met `UNAVAILABLE` in plaats van te doen alsof de lay-out is gewijzigd.

  </Accordion>

  <Accordion title="Automatisering, Skills en hulpmiddelen">
    - Automatisering: `wake` plant een onmiddellijke of bij-de-volgende-Heartbeat uitgevoerde injectie van activeringstekst; `cron.get`, `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`, `cron.run` en `cron.runs` beheren gepland werk.
    - `cron.run` blijft een RPC volgens het wachtrijmodel voor handmatige uitvoeringen. Clients die voltooiingssemantiek nodig hebben, moeten de geretourneerde `runId` lezen en `cron.runs` pollen.
    - `cron.runs` accepteert een optioneel niet-leeg `runId`-filter, zodat clients één handmatig in de wachtrij geplaatste uitvoering kunnen volgen zonder te concurreren met andere geschiedenisvermeldingen voor dezelfde taak.
    - Skills en hulpmiddelen: `commands.list`, `skills.*`, `tools.catalog`, `tools.effective`, `tools.invoke`. Zie hieronder [Hulpmethoden voor operators](#operator-helper-methods).

  </Accordion>
</AccordionGroup>

### Algemene gebeurteniscategorieën

- `chat`: UI-chatupdates zoals `chat.inject` en andere chatgebeurtenissen
  die alleen in het transcript voorkomen. In protocol v4 bevatten delta-payloads `deltaText`; `message` blijft
  de cumulatieve momentopname van de assistent. Vervangingen die geen prefix zijn, stellen
  `replace=true` in en gebruiken `deltaText` als vervangende tekst.
- `session.message`, `session.operation`, `session.tool`: updates voor het transcript, een lopende
  sessiebewerking en de gebeurtenisstroom van een gevolgde sessie.
- `session.approval`: opgeschoonde waarheid over openstaande en definitieve goedkeuringen voor een
  expliciet aangemelde abonnee van exact die sessie. Onderliggende goedkeuringen gebruiken het
  opgeslagen publiek van de bovenliggende goedkeuring; gebeurtenissen wijzigen nooit transcripten en activeren geen agents.
- `sessions.changed`: sessie-index of metagegevens gewijzigd.
- `presence`: updates van de momentopname van systeemaanwezigheid.
- `tick`: periodieke keepalive-/levendigheidsgebeurtenis.
- `health`: update van de momentopname van de Gateway-status.
- `heartbeat`: update van de Heartbeat-gebeurtenisstroom.
- `cron`: gebeurtenis bij een wijziging van een Cron-uitvoering of -taak.
- `shutdown`: melding dat de Gateway wordt afgesloten.
- `node.pair.requested` / `node.pair.resolved`: levenscyclus van nodekoppeling.
- `node.invoke.request`: uitzending van een node-aanroepaanvraag.
- `device.pair.requested` / `device.pair.resolved`: levenscyclus van gekoppelde apparaten.
- `voicewake.changed`: configuratie voor activering via een trefwoord gewijzigd.
- `config.changed`: een configuratieschrijfactie is opgeslagen (de payload bevat het configuratiepad,
  de hash van de nieuwe momentopname en een tijdstempel — nooit configuratie-inhoud). Beperkt
  tot operators met leesbereik; clients vernieuwen via `config.get`.
- `exec.approval.requested` / `exec.approval.resolved`: levenscyclus van
  uitvoeringsgoedkeuringen.
- `plugin.approval.requested` / `plugin.approval.resolved`: levenscyclus van
  plugingoedkeuringen.

### Hulpmethoden voor nodes

Nodes kunnen `skills.bins` aanroepen om de huidige lijst met uitvoerbare Skill-bestanden
op te halen voor controles op automatische toestemming.

## RPC voor het auditlogboek

`audit.activity.list` biedt operatorclients een stabiele weergave, met de nieuwste eerst, van metagegevens over de levenscyclus van
agentuitvoeringen, hulpmiddelacties en vrijwillig opgenomen berichten. Hiervoor is
`operator.read` vereist. Query's sluiten records ouder dan 30 dagen uit en het gedeelde
SQLite-logboek is beperkt tot 100.000 records. Verlopen rijen worden verwijderd tijdens
het opstarten van de Gateway, bij elk uuronderhoud en bij latere schrijfacties. Zie
[Auditgeschiedenis](/nl/gateway/audit) voor het gegevensmodel en de privacysemantiek.

- Parameters: optioneel exact `agentId`, `sessionKey` of `runId`; optioneel `kind`
  (`"agent_run"`, `"tool_action"` of `"message"`); optioneel `status`
  (`"started"`, `"succeeded"`, `"failed"`, `"cancelled"`, `"timed_out"`,
  `"blocked"` of `"unknown"`); optioneel bericht-`direction` (`"inbound"` of
  `"outbound"`) en exact `channel`; optionele inclusieve Unix-millisecondegrenzen `after` / `before`;
  optioneel `limit` van `1` tot `500`; en optionele tekenreeks
  `cursor` van de voorgaande pagina.
- Resultaat: `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`.

De benoemde V1-resultaatunie heeft afzonderlijke schema's voor agentuitvoeringen, hulpmiddelacties, inkomende berichten
en uitgaande berichten. De discriminator `eventType` is respectievelijk
`agent_run`, `tool_action`, `inbound_message` of `outbound_message`; `kind` en
bericht-`direction` blijven beschikbaar voor filtering en weergave. Elke gebeurtenis heeft
een gehele `schemaVersion: 1`. Verwijzingen naar berichtidentiteiten gebruiken de exacte
`hmac-sha256:v1:<32 hex key id>:<64 hex digest>`-indeling; een actor-id van een kanaalafzender
gebruikt dezelfde indeling.

Alle varianten vereisen `eventType`, `schemaVersion`, `eventId`, `sequence`,
`sourceSequence`, `occurredAt`, `kind`, `action`, `status`, `actor` en
`redaction`. De variantvelden zijn:

| `eventType`        | Vereiste velden                                                   | Optionele velden                                                                                                                 |
| ------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `agent_run`        | `agentId`, `runId`; `kind: "agent_run"`                           | `sessionKey`, `sessionId`, `errorCode`                                                                                          |
| `tool_action`      | `agentId`, `runId`; `kind: "tool_action"`                         | `sessionKey`, `sessionId`, `toolCallId`, `toolName`, `errorCode`                                                                |
| `inbound_message`  | `direction: "inbound"`, `channel`, `conversationKind`, `outcome`  | `agentId`, `runId`, `durationMs`, `resultCount`, identiteitsverwijzingen, `reasonCode`, `errorCode`                                 |
| `outbound_message` | `direction: "outbound"`, `channel`, `conversationKind`, `outcome` | `agentId`, `runId`, `durationMs`, `resultCount`, identiteitsverwijzingen, `reasonCode`, `deliveryKind`, `failureStage`, `errorCode` |

De gesloten berichten-enums zijn:

- `conversationKind`: `direct`, `group`, `channel` of `unknown`.
- Inkomend `outcome`: `completed`, `skipped` of `failed`; optioneel
  `reasonCode`: `duplicate`, `reply_operation_active`,
  `reply_operation_aborted`, `fast_abort`, `plugin_bound_handled`,
  `plugin_bound_unavailable`, `plugin_bound_declined`, `plugin_bound_error`,
  `before_dispatch_handled`, `acp_dispatch_completed`, `acp_dispatch_failed`,
  `acp_dispatch_empty` of `acp_dispatch_aborted`.
- Uitgaand `outcome`: `sent`, `suppressed`, `failed` of `unknown`; optioneel
  `reasonCode`: `cancelled_by_message_sending_hook`,
  `cancelled_by_reply_payload_sending_hook`,
  `empty_after_message_sending_hook`, `empty_after_reply_payload_sending_hook`
  of `no_visible_payload`. Een adapter die geen platformidentiteit retourneert, is
  `unknown`, omdat de externe bijwerking niet kan worden weerlegd.
- `deliveryKind`: `text`, `media` of `other`; `failureStage`:
  `platform_send`, `queue` of `unknown`.

Definitieve velden zijn gecorreleerd en niet onafhankelijk optioneel:

| Variant          | Definitieve toewijzing                                                                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Agentuitvoering        | `started` heeft geen `errorCode`; elke voltooide status die geen succes aangeeft, vereist de bijbehorende `run_*`-code.                                                                 |
| Hulpmiddelactie      | `started` en geslaagd hebben geen `errorCode`; elke andere voltooide status vereist de bijbehorende `tool_*`-code.                                                       |
| Inkomend bericht  | geslaagd = `completed`; geblokkeerd = `skipped`; mislukt = `failed` plus `message_processing_failed`. `reasonCode` moet, indien aanwezig, tot die definitieve categorie behoren. |
| Uitgaand bericht | geslaagd = `sent`; geblokkeerd = `suppressed` plus `reasonCode`; mislukt = `failed` plus `errorCode` en `failureStage`; onbekend = `unknown` plus `failureStage`.      |

Elke activiteitsgebeurtenis bevat een stabiele gebeurtenis-id, monotoon oplopend logboekvolgnummer,
brongebeurtenisvolgnummer, tijdstempel, actor, actie, status, gehele
`schemaVersion: 1` en `redaction: "metadata_only"`. Uitvoerings- en hulpmiddelrecords
vereisen de herkomst van de agent en uitvoering en kunnen sessieherkomst bevatten. Berichtrecords
kunnen agent- en uitvoerings-id's bevatten, maar bevatten bewust nooit
`sessionKey` of `sessionId`; het queryfilter `sessionKey` is daarom alleen van toepassing op
uitvoerings- en hulpmiddelrijen. Hulpmiddelgebeurtenissen kunnen de id van de hulpmiddelaanroep en de hulpmiddelnaam bevatten.

Berichtrecords gebruiken `message.inbound.processed` of
`message.outbound.finished` en voegen richting, kanaal, gesprekstype,
genormaliseerd resultaat en optioneel afleveringstype, foutfase, duur,
aantal resultaten, redencode en met een installatiegebonden sleutel aangemaakte
pseudoniemen voor account/gesprek/bericht/doel toe. Deze pseudoniemen helpen bij
correlatie, maar zijn geen anonimisering: de statusdatabase bevat hun sleutel,
terwijl RPC- en CLI-exports die niet bevatten. Het logboek slaat geen prompts, berichtinhoud,
toolargumenten, toolresultaten, opdrachtuitvoer of onbewerkte fouttekst op.
Run/tool-waarden van `sessionKey` blijven onbewerkte correlatiemetadata en kunnen
platformaccount- of peer-id's bevatten; berichtrecords bevatten geen sessiesleutels.

Voor inkomende rijen meet `durationMs` de kerndispatch tot en met de eindstatus en
telt `resultCount` de definitief verwerkte tool-, blok- en antwoordpayloads in de wachtrij. Voor
uitgaande rijen omvat `durationMs` het eigenaarschap van de aflevering tot en met bevestiging,
dead letter of reconciliatie (inclusief wachttijd in de wachtrij), en telt `resultCount`
de geïdentificeerde fysieke verzendingen naar het platform. `deliveryKind` beschrijft, indien aanwezig,
de effectieve payload na hooks en rendering; onderdrukte rijen of rijen
waarvan de status door een crash onduidelijk is, bevatten deze waarde niet.

De huidige berichtdekking omvat geaccepteerde inkomende berichten die de
kerndispatch bereiken, inclusief dubbele/eindresultaten van de kern. Voor uitgaande berichten wordt
één eindrij geschreven per oorspronkelijke logische antwoordpayload die de gedeelde duurzame
aflevering bereikt; segmentering en adapter-fan-out worden samengevoegd in `resultCount`. Opnieuw
te proberen verzendingen in de wachtrij of verzendingen met een onduidelijke status worden pas vastgelegd na bevestiging, een
dead letter of reconciliatie. Plugin-lokale en directe verzendpaden die deze
gedeelde grenzen omzeilen, worden nog niet gedekt. De begrensde workerwachtrij werkt volgens het best-effortprincipe
en kan records laten vallen bij fouten of verzadiging. Dit oppervlak is daarom geen
verliesvrij nalevingsarchief.

Registratie is standaard ingeschakeld en wordt beheerd via
[`audit.enabled`](/nl/gateway/configuration-reference#audit). Berichtregistratie wordt
afzonderlijk beheerd via `audit.messages` en is standaard ingesteld op `"off"`. Wanneer
registratie is uitgeschakeld, blijft `audit.activity.list` eerder geschreven records aanbieden
totdat ze verlopen.

De meegeleverde schema's voor het `audit.list`-verzoek, het resultaat en `AuditEvent`
blijven ongewijzigd en retourneren alleen records van agentruns en toolacties. Nieuwe operatorclients
moeten `audit.activity.list` aanroepen wanneer de Gateway deze methode adverteert. Oudere
Gateways kunnen `unknown method: audit.activity.list` rapporteren of, omdat
autorisatie in meegeleverde versies voorafging aan het opzoeken van de methode, `missing scope:
operator.admin` voor een verzoek met leesbereik. Behandel dat laatste alleen als
afwezigheid van de methode wanneer de methode niet was geadverteerd. Een client kan vervolgens alleen `audit.list`
opnieuw proberen wanneer de filters geen ondersteuning voor berichtstype, richting of kanaal
vereisen.

Gebruik [`openclaw audit`](/nl/cli/audit) voor tekstquery's en begrensde JSON-exports.

## RPC's voor het taaklogboek

Operatorclients inspecteren en annuleren records van achtergrondtaken van de Gateway via
de RPC's van het taaklogboek (`packages/gateway-protocol/src/schema/tasks.ts`). Deze
retourneren opgeschoonde taaksamenvattingen, geen onbewerkte runtimestatus.

- `tasks.list` vereist `operator.read`.
  - Parameters: optioneel `status` (`"queued"`, `"running"`, `"completed"`,
    `"failed"`, `"cancelled"` of `"timed_out"`) of een array met deze statussen,
    optioneel `agentId`, optioneel `sessionKey`, optioneel `limit` van `1` tot
    `500` en optionele tekenreeks `cursor`.
  - Resultaat: `{ "tasks": TaskSummary[], "nextCursor"?: string }`.
- `tasks.get` vereist `operator.read`.
  - Parameters: `{ "taskId": string }`.
  - Resultaat: `{ "task": TaskSummary }`.
  - Ontbrekende taak-id's retourneren de vorm van de niet-gevonden-fout van de Gateway.
- `tasks.cancel` vereist `operator.write`.
  - Parameters: `{ "taskId": string, "reason"?: string }`.
  - Resultaat: `{ "found": boolean, "cancelled": boolean, "reason"?: string, "task"?: TaskSummary }`.
  - `found` geeft aan of het logboek een overeenkomende taak bevatte. `cancelled`
    geeft aan of de runtime de annulering heeft geaccepteerd of vastgelegd.

`TaskSummary` bevat `id`, `status` en optionele metadata: `kind`,
`runtime`, `title`, `agentId`, `sessionKey`, `childSessionKey`, `ownerKey`,
`runId`, `taskId`, `flowId`, `parentTaskId`, `sourceId`, tijdstempels, voortgang,
eindsamenvatting en opgeschoonde fouttekst. `agentId` identificeert de agent
die de taak uitvoert; `sessionKey` en `ownerKey` behouden de context van de aanvrager en de besturing.

## Hulpmethoden voor operators

- `commands.list` (`operator.read`) haalt de runtimeopdrachtinventaris voor
  een agent op.
  - `agentId` is optioneel; laat dit weg om de standaardwerkruimte van de agent te lezen.
  - `scope` bepaalt op welk oppervlak de primaire `name` is gericht: `text` retourneert
    het primaire token van de tekstopdracht zonder de voorafgaande `/`; `native` en het
    standaardpad `both` retourneren providerbewuste systeemeigen namen wanneer die beschikbaar zijn.
  - `textAliases` bevat exacte slash-aliassen zoals `/model` en `/m`.
  - `nativeName` bevat de providerbewuste systeemeigen opdrachtnaam wanneer die
    bestaat.
  - `provider` is optioneel en beïnvloedt alleen systeemeigen naamgeving en de beschikbaarheid van systeemeigen Plugin-
    opdrachten.
  - `includeArgs=false` laat geserialiseerde argumentmetadata weg uit het antwoord.
- `tools.catalog` (`operator.read`) haalt de runtimetoolcatalogus voor een
  agent op. Het antwoord bevat gegroepeerde tools en herkomstmetadata:
  - `source`: `core` of `plugin`
  - `pluginId`: eigenaar van de Plugin wanneer `source="plugin"`
  - `optional`: of een Plugin-tool optioneel is
- `tools.effective` (`operator.read`) haalt de runtime-effectieve toolinventaris
  voor een sessie op.
  - `sessionKey` is vereist.
  - De Gateway leidt vertrouwde runtimecontext server-side af uit de sessie
    in plaats van door de aanroeper verstrekte authenticatie- of afleveringscontext te accepteren.
  - Het antwoord is een sessiegebonden, door de server afgeleide projectie van de actieve
    inventaris, inclusief tools van de kern, Plugins, kanalen en reeds ontdekte MCP-
    servers.
  - `tools.effective` is alleen-lezen voor MCP: deze methode kan een MCP-
    catalogus van een warme sessie via het definitieve toolbeleid projecteren, maar maakt geen MCP-runtimes,
    verbindt geen transports en geeft geen `tools/list` uit. Als er geen overeenkomende warme catalogus
    bestaat, kan het antwoord een melding bevatten zoals `mcp-not-yet-connected`,
    `mcp-not-yet-listed` of `mcp-stale-catalog`.
  - Effectieve toolvermeldingen gebruiken `source="core"`, `source="plugin"`,
    `source="channel"` of `source="mcp"`.
- `tools.invoke` (`operator.write`) roept één beschikbare tool aan via hetzelfde
  Gateway-beleidspad als `/tools/invoke`.
  - `name` is vereist. `args`, `sessionKey`, `agentId`, `confirm` en
    `idempotencyKey` zijn optioneel.
  - Als zowel `sessionKey` als `agentId` aanwezig zijn, moet de opgeloste sessieagent
    overeenkomen met `agentId`.
  - Kernwrappers die alleen voor de eigenaar zijn bedoeld, zoals `cron`, `gateway` en `nodes`, vereisen
    een eigenaars-/beheerdersidentiteit (`operator.admin`), hoewel `tools.invoke` zelf
    `operator.write` is.
  - Het antwoord is een op de SDK gerichte envelop met `ok`, `toolName`, optioneel
    `output` en getypeerde `error`-velden. Weigeringen wegens goedkeuring of beleid retourneren
    `ok:false` in de payload in plaats van de Gateway-pijplijn voor
    toolbeleid te omzeilen.
- `skills.status` (`operator.read`) haalt de zichtbare Skills-inventaris voor een
  agent op.
  - `agentId` is optioneel; laat dit weg om de standaardwerkruimte van de agent te lezen.
  - Het antwoord bevat geschiktheid, ontbrekende vereisten, configuratiecontroles
    en opgeschoonde installatieopties zonder onbewerkte geheime waarden bloot te stellen.
- `skills.search` en `skills.detail` (`operator.read`) retourneren ClawHub-
  detectiemetadata.
- `skills.upload.begin`, `skills.upload.chunk` en `skills.upload.commit`
  (`operator.admin`) bereiden een privé-Skills-archief voor voordat het wordt geïnstalleerd. Dit
  is een afzonderlijk uploadpad voor beheerders voor vertrouwde clients, niet de normale ClawHub-
  installatiestroom voor Skills, en is standaard uitgeschakeld tenzij
  `skills.install.allowUploadedArchives` is ingeschakeld.
  - `skills.upload.begin({ kind: "skill-archive", slug, sizeBytes, sha256?, force?, idempotencyKey? })`
    maakt een upload die aan die slug en force-waarde is gekoppeld.
  - `skills.upload.chunk({ uploadId, offset, dataBase64 })` voegt bytes toe op
    de exacte gedecodeerde offset.
  - `skills.upload.commit({ uploadId, sha256? })` verifieert de uiteindelijke grootte en
    SHA-256. Commit voltooit alleen de upload; de Skill wordt niet geïnstalleerd.
  - Geüploade Skills-archieven zijn ziparchieven met een `SKILL.md`-hoofdmap. De
    interne mapnaam van het archief bepaalt nooit het installatiedoel.
- `skills.install` (`operator.admin`) heeft drie modi:
  - ClawHub-modus: `{ source: "clawhub", slug, version?, force? }` installeert een
    Skills-map in de map `skills/` van de standaardwerkruimte van de agent.
  - Uploadmodus: `{ source: "upload", uploadId, slug, force?, sha256?, timeoutMs? }`
    installeert een vastgelegde upload in de map
    `skills/<slug>` van de standaardwerkruimte van de agent. De slug en force-waarde moeten overeenkomen met het
    oorspronkelijke `skills.upload.begin`-verzoek. Wordt geweigerd tenzij
    `skills.install.allowUploadedArchives` is ingeschakeld; de instelling heeft geen
    invloed op ClawHub-installaties.
  - Gateway-installatiemodus: `{ name, installId, timeoutMs? }` voert een gedeclareerde
    `metadata.openclaw.install`-actie uit op de Gateway-host. Oudere clients kunnen
    nog steeds `dangerouslyForceUnsafeInstall` verzenden; dit veld is verouderd,
    wordt alleen geaccepteerd voor protocolcompatibiliteit en wordt genegeerd. Gebruik
    `security.installPolicy` voor installatiebeslissingen van operators.
- `skills.update` (`operator.admin`) heeft twee modi:
  - De ClawHub-modus werkt één bijgehouden slug of alle bijgehouden ClawHub-installaties in
    de standaardwerkruimte van de agent bij.
  - De configuratiemodus past `skills.entries.<skillKey>`-waarden aan, zoals `enabled`,
    `apiKey` en `env`.

### `models.list`-weergaven

`models.list` accepteert een optionele parameter `view`
(`src/agents/model-catalog-visibility.ts`):

- Weggelaten of `"default"`: als `agents.defaults.modelPolicy.allow` is geconfigureerd, is het
  antwoord de toegestane catalogus, inclusief dynamisch ontdekte modellen
  voor `provider/*`-vermeldingen. Anders is het antwoord de volledige Gateway-
  catalogus.
- `"configured"`: gedrag met de omvang van een keuzelijst. Als `agents.defaults.modelPolicy.allow` is
  geconfigureerd, krijgt deze nog steeds voorrang, inclusief providergebonden detectie voor
  `provider/*`-vermeldingen. Zonder een toelatingslijst gebruikt het antwoord expliciete
  `models.providers.<provider>.models`-vermeldingen, met een terugval op de volledige
  catalogus alleen wanneer er geen geconfigureerde modelrijen bestaan.
- `"provider-config"`: door de bron opgestelde `models.providers.*.models`-inventaris,
  onafhankelijk van toelatingslijsten voor keuzelijsten. Rijen bevatten openbare modelmogelijkheden en
  routebewuste beschikbaarheid, maar laten providereindpunten, authenticatiemateriaal en
  runtimeverzoekconfiguratie weg.
- `"all"`: volledige Gateway-catalogus, waarbij `agents.defaults.modelPolicy.allow` wordt omzeild. Gebruik dit voor
  diagnostische/detectie-UI's, niet voor normale modelkeuzelijsten.

## Uitvoeringsgoedkeuringen

- Wanneer een exec-verzoek goedkeuring vereist, zendt de Gateway
  `exec.approval.requested` uit.
- Operatorclients handelen dit af door `exec.approval.resolve` aan te roepen (vereist
  `operator.approvals`).
- Voor `host=node` moet `exec.approval.request` `systemRunPlan` bevatten
  (canonieke `argv`/`cwd`/`rawCommand`/sessiemetadata). Verzoeken zonder
  `systemRunPlan` worden geweigerd.
- Na goedkeuring hergebruiken doorgestuurde `node.invoke system.run`-aanroepen die
  canonieke `systemRunPlan` als de gezaghebbende opdracht-/cwd-/sessiecontext.
- Als een aanroeper `command`, `rawCommand`, `cwd`, `agentId` of
  `sessionKey` wijzigt tussen de voorbereiding en het uiteindelijk goedgekeurde doorsturen van `system.run`,
  weigert de Gateway de uitvoering in plaats van de gewijzigde payload te vertrouwen.

## Terugvaloptie voor agentbezorging

- `agent`-verzoeken kunnen `deliver=true` bevatten om uitgaande bezorging aan te vragen.
- `bestEffortDeliver=false` (de standaardwaarde) handhaaft strikt gedrag: niet-opgeloste of
  uitsluitend interne bezorgingsdoelen retourneren `INVALID_REQUEST`.
- `bestEffortDeliver=true` staat terugvallen op uitvoering binnen alleen de sessie toe wanneer geen
  externe bezorgbare route kan worden bepaald (bijvoorbeeld bij interne/webchat-
  sessies of dubbelzinnige configuraties met meerdere kanalen).
- Uiteindelijke `agent`-resultaten kunnen `result.deliveryStatus` bevatten wanneer bezorging is
  aangevraagd, met dezelfde statussen `sent`, `suppressed`, `partial_failed` en
  `failed` die zijn gedocumenteerd voor
  [`openclaw agent --json --deliver`](/nl/cli/agent#json-delivery-status).

## Versiebeheer

- `PROTOCOL_VERSION`, `MIN_CLIENT_PROTOCOL_VERSION`,
  `MIN_NODE_PROTOCOL_VERSION` en `MIN_PROBE_PROTOCOL_VERSION` bevinden zich in
  `packages/gateway-protocol/src/version.ts`.
- Clients verzenden `minProtocol` + `maxProtocol`. Operator- en UI-clients moeten
  het huidige protocol in dat bereik opnemen; huidige clients en servers gebruiken
  protocol v4.
- Geverifieerde clients met zowel `role: "node"` als `client.mode: "node"`
  mogen het N-1-nodeprotocol gebruiken (momenteel v3). Lichtgewicht herstartcontroles gebruiken
  hetzelfde N-1-venster. Apparaatverificatie, koppeling, bereiken, opdrachtbeleid en exec-
  goedkeuringen blijven door dit compatibiliteitsvenster ongewijzigd. Door Plugins beheerde node-
  mogelijkheden en opdrachten worden niet beschikbaar gesteld totdat de node naar het huidige
  protocol is bijgewerkt, omdat de door hen gehoste oppervlakken geen deel uitmaken van het N-1-contract.
- Schema's en modellen worden gegenereerd uit TypeBox-definities:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### Clientconstanten

De referentie-implementatie van de client bevindt zich in `packages/gateway-client/src/`
(OpenClaw verpakt deze via de dunne `src/gateway/client.ts`-facade). Deze
standaardwaarden zijn stabiel in protocol v4 en vormen de verwachte basis voor
clients van derden.

| Constante                                 | Standaardwaarde                                       | Bron                                                                                                                      |
| ----------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `PROTOCOL_VERSION`                        | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_CLIENT_PROTOCOL_VERSION`             | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_NODE_PROTOCOL_VERSION`               | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_PROBE_PROTOCOL_VERSION`              | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| Time-out van verzoek (per RPC)            | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`requestTimeoutMs`)                                                              |
| Time-out voor preauth/verbindingsuitdaging | `15_000` ms                                           | `packages/gateway-client/src/timeouts.ts` (env `OPENCLAW_HANDSHAKE_TIMEOUT_MS` kan het budget van de gekoppelde server/client verhogen) |
| Initiële back-off voor opnieuw verbinden  | `1_000` ms                                            | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Maximale back-off voor opnieuw verbinden  | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Begrenzing voor snelle nieuwe poging na sluiting wegens apparaattoken | `250` ms                                              | `packages/gateway-client/src/client.ts`                                                                                   |
| Respijtperiode voor geforceerd stoppen vóór `terminate()` | `250` ms                                              | `FORCE_STOP_TERMINATE_GRACE_MS`                                                                                           |
| Standaardtime-out voor `stopAndWait()` | `1_000` ms                                            | `STOP_AND_WAIT_TIMEOUT_MS`                                                                                                |
| Standaard tick-interval (vóór `hello-ok`) | `30_000` ms                                           | `packages/gateway-client/src/client.ts`                                                                                   |
| Sluiting wegens tick-time-out             | code `4000` wanneer de stilte langer duurt dan `tickIntervalMs * 2` | `packages/gateway-client/src/client.ts`                                                                                   |
| `MAX_PAYLOAD_BYTES`                       | `25 * 1024 * 1024` (25 MB)                            | `src/gateway/server-constants.ts`                                                                                         |

De server maakt de effectieve `policy.tickIntervalMs`,
`policy.maxPayload` en `policy.maxBufferedBytes` bekend in `hello-ok`; clients
moeten die waarden respecteren in plaats van de standaardwaarden van vóór de handshake.

De referentieclient laat eindige verzoeken hun geconfigureerde deadline beheren wanneer
elk verzoek in behandeling er een heeft. Een `expectFinal`-verzoek zonder een eindige
`timeoutMs`, elk verzoek met `timeoutMs: null` of een combinatie van eindige en
onbegrensde verzoeken houdt de tick-watchdog actief. Als inkomende gebeurtenissen en
antwoorden langer stil blijven dan de drempel voor de tick-time-out, sluit de client de
socket met code `4000`, wijst elk verzoek in behandeling af en maakt opnieuw verbinding. De client
speelt afgewezen verzoeken na het opnieuw verbinden niet opnieuw af.

## Verificatie

- Gateway-authenticatie met een gedeeld geheim gebruikt `connect.params.auth.token` of
  `connect.params.auth.password`, afhankelijk van de geconfigureerde
  `gateway.auth.mode` (`"none" | "token" | "password" | "trusted-proxy"`).
- Modi die identiteit bevatten, zoals Tailscale Serve (`gateway.auth.allowTailscale: true`)
  of niet-loopback-`gateway.auth.mode: "trusted-proxy"`, voldoen op basis van aanvraagheaders aan de
  authenticatiecontrole voor de verbinding, in plaats van via `connect.params.auth.*`.
- `gateway.auth.mode: "none"` voor privé-ingang slaat authenticatie met een gedeeld geheim bij het verbinden
  volledig over; stel die modus niet beschikbaar via een openbare/niet-vertrouwde ingang.
- Na het koppelen geeft de Gateway een apparaattoken uit dat beperkt is tot de
  verbindingsrol en -bereiken en wordt geretourneerd in `hello-ok.auth.deviceToken`. Clients moeten
  dit na elke geslaagde verbinding opslaan.
- Bij opnieuw verbinden met dat opgeslagen apparaattoken moet ook de opgeslagen
  goedgekeurde bereikenset voor dat token opnieuw worden gebruikt. Dit behoudt reeds
  verleende lees-/controle-/statustoegang en voorkomt dat nieuwe verbindingen ongemerkt
  worden beperkt tot een smaller impliciet bereik dat alleen voor beheerders bestemd is.
- Samenstelling van verbindingsauthenticatie aan clientzijde (`selectConnectAuth` in
  `packages/gateway-client/src/client.ts`):
  - `auth.password` staat hier los van en wordt altijd doorgestuurd wanneer het is ingesteld.
  - `auth.token` wordt in deze prioriteitsvolgorde ingevuld: eerst een expliciet gedeeld token,
    vervolgens een expliciete `deviceToken` en daarna een opgeslagen token per apparaat (met
    `deviceId` + `role` als sleutel).
  - `auth.bootstrapToken` wordt alleen verzonden wanneer geen van de bovenstaande opties
    `auth.token` heeft opgeleverd. Een gedeeld token of elk gevonden apparaattoken onderdrukt dit.
  - Automatische promotie van een opgeslagen apparaattoken bij de eenmalige
    nieuwe poging voor `AUTH_TOKEN_MISMATCH` is uitsluitend toegestaan voor vertrouwde eindpunten: loopback
    of `wss://` met een vastgezette `tlsFingerprint`. Openbare `wss://` zonder vastzetten
    komt niet in aanmerking.
- De ingebouwde bootstrap met installatiecode retourneert de `hello-ok.auth.deviceToken` van de primaire Node
  plus een begrensd operatortoken in
  `hello-ok.auth.deviceTokens` voor vertrouwde overdracht naar mobiele apparaten. Het operatortoken
  bevat `operator.talk.secrets` voor het lezen van de native Talk-configuratie, maar
  sluit bereiken voor koppelingswijzigingen en `operator.admin` uit.
- Terwijl een bootstrap met een niet-standaardinstallatiecode op goedkeuring wacht,
  bevatten de details van `PAIRING_REQUIRED` `recommendedNextStep: "wait_then_retry"`,
  `retryable: true` en `pauseReconnect: false`. Blijf opnieuw verbinden met hetzelfde
  bootstraptoken totdat de aanvraag is goedgekeurd of het token ongeldig wordt.
- Sla `hello-ok.auth.deviceTokens` alleen op wanneer voor de verbinding bootstrap-
  authenticatie via een vertrouwd transport is gebruikt, zoals `wss://` of lokale koppeling/loopback.
- Als een client een expliciete `deviceToken` of expliciete `scopes` opgeeft, blijft die
  door de aanroeper aangevraagde bereikenset leidend; gecachte bereiken worden alleen
  hergebruikt wanneer de client het opgeslagen token per apparaat hergebruikt.
- Apparaattokens kunnen worden geroteerd/ingetrokken via `device.token.rotate` en
  `device.token.revoke` (vereist `operator.pairing`). Voor het roteren of intrekken van een
  Node of een andere niet-operatorrol is ook `operator.admin` vereist.
- `device.token.rotate` retourneert rotatiemetadata. Het geeft het vervangende
  bearertoken alleen terug bij aanroepen vanaf hetzelfde apparaat die al met dat
  apparaattoken zijn geauthenticeerd, zodat clients die uitsluitend tokens gebruiken hun vervanging
  vóór het opnieuw verbinden kunnen opslaan. Rotaties door middel van gedeelde/beheerderstokens geven het bearertoken niet terug.
- Uitgifte, rotatie en intrekking van tokens blijven beperkt tot de goedgekeurde
  rollenset die in de koppelingsvermelding van dat apparaat is vastgelegd; tokenwijzigingen kunnen geen
  apparaatrol uitbreiden of als doel kiezen die nooit via koppelingsgoedkeuring is verleend.
- Voor tokensessies van gekoppelde apparaten is apparaatbeheer beperkt tot het eigen apparaat, tenzij
  de aanroeper ook `operator.admin` heeft: niet-beheerders kunnen alleen het
  operatortoken voor hun eigen apparaatvermelding beheren. Tokenbeheer voor Nodes en andere
  niet-operatorrollen is uitsluitend voor beheerders, zelfs voor het eigen apparaat van de aanroeper.
- `device.token.rotate` en `device.token.revoke` controleren ook de bereikenset van het beoogde
  operatortoken aan de hand van de huidige sessiebereiken van de aanroeper.
  Niet-beheerders kunnen geen operatortoken roteren of intrekken dat ruimere bereiken heeft dan
  het token waarover ze al beschikken.
- Authenticatiefouten bevatten `error.details.code` plus hersteltips:
  - `error.details.canRetryWithDeviceToken` (booleaans)
  - `error.details.recommendedNextStep`: een van `retry_with_device_token`,
    `update_auth_configuration`, `update_auth_credentials`,
    `wait_then_retry`, `review_auth_configuration`
    (`packages/gateway-protocol/src/connect-error-details.ts`).
- Clientgedrag voor `AUTH_TOKEN_MISMATCH`:
  - Vertrouwde clients mogen één begrensde nieuwe poging doen met een gecacht token per apparaat.
  - Als die nieuwe poging mislukt, stop dan automatische lussen voor opnieuw verbinden en toon
    aanwijzingen voor actie door de operator.
- `AUTH_SCOPE_MISMATCH` betekent dat het apparaattoken is herkend, maar niet
  de aangevraagde rol/bereiken dekt. Presenteer dit niet als een ongeldig token; vraag
  de operator opnieuw te koppelen of het smallere/ruimere bereikcontract goed te keuren.

## Apparaatidentiteit en koppeling

- Nodes moeten een stabiele apparaatidentiteit (`device.id`) bevatten die is afgeleid van de
  vingerafdruk van een sleutelpaar.
- Gateways geven tokens uit per apparaat + rol.
- Voor nieuwe apparaat-ID's is koppelingsgoedkeuring vereist, tenzij lokale
  automatische goedkeuring is ingeschakeld.
- Automatische koppelingsgoedkeuring is gericht op directe lokale loopbackverbindingen.
- OpenClaw heeft ook een beperkt zelfverbindingspad dat lokaal is voor de backend/container,
  voor vertrouwde helperstromen met een gedeeld geheim.
- Tailnet- of LAN-verbindingen vanaf dezelfde host worden voor koppeling nog steeds als extern behandeld
  en vereisen goedkeuring.
- WS-clients nemen tijdens `connect` normaal gesproken de identiteit `device` op (operator +
  Node). De enige uitzonderingen voor operators zonder apparaat zijn expliciete vertrouwenspaden:
  - geslaagde `gateway.auth.mode: "trusted-proxy"`-operatorauthenticatie voor de Control UI.
  - directe loopback-`gateway-client`-backend-RPC's op het gereserveerde interne
    helperpad.
- Het weglaten van de apparaatidentiteit heeft gevolgen voor de bereiken. Wanneer een operatorverbinding
  zonder apparaat via een expliciet vertrouwenspad wordt toegestaan, wist OpenClaw
  zelf opgegeven bereiken nog steeds tot een lege set, tenzij dat pad een
  benoemde uitzondering voor bereikbehoud heeft. Methoden waarvoor bereiken vereist zijn, mislukken vervolgens met
  `missing scope`.
- Het gereserveerde directe loopback-`gateway-client`-backendhelperpad behoudt
  bereiken alleen voor interne lokale RPC's van het besturingsvlak; aangepaste backend-ID's
  krijgen deze uitzondering niet.
- Alle verbindingen moeten de door de server verstrekte `connect.challenge`-nonce ondertekenen.

### Migratiediagnostiek voor apparaatauthenticatie

Voor verouderde clients die nog ondertekeningsgedrag van vóór de challenge gebruiken, retourneert `connect`
`DEVICE_AUTH_*`-detailcodes onder `error.details.code` met een stabiele
`error.details.reason`.

Veelvoorkomende migratiefouten:

| Bericht                     | details.code                     | details.reason           | Betekenis                                          |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Client liet `device.nonce` weg (of verzond een lege waarde). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Client ondertekende met een verouderde/verkeerde nonce. |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | De ondertekeningspayload komt niet overeen met de v2-payload. |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | Het ondertekende tijdstempel valt buiten de toegestane afwijking. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` komt niet overeen met de vingerafdruk van de openbare sleutel. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | De indeling/canonicalisatie van de openbare sleutel is mislukt. |

Migratiedoel:

- Wacht altijd op `connect.challenge`.
- Onderteken de v2-payload die de servernonce bevat.
- Verzend dezelfde nonce in `connect.params.device.nonce`.
- De voorkeurspayload voor ondertekening is `v3`
  (`buildDeviceAuthPayloadV3` in `packages/gateway-client/src/device-auth.ts`),
  die naast de velden voor apparaat/client/rol/bereiken/token/nonce ook
  `platform` en `deviceFamily` bindt.
- Verouderde `v2`-handtekeningen blijven voor compatibiliteit geaccepteerd, maar het vastzetten
  van metadata van gekoppelde apparaten blijft bij opnieuw verbinden het opdrachtenbeleid bepalen.

## TLS en vastzetten

- TLS wordt ondersteund voor WS-verbindingen (`gateway.tls`-configuratie).
- Clients kunnen optioneel de vingerafdruk van het Gateway-certificaat vastzetten via
  `gateway.remote.tlsFingerprint` of CLI `--tls-fingerprint`.

## Bereik

Dit protocol ontsluit de volledige Gateway-API: status, kanalen, modellen, chat,
agent, sessies, Nodes, goedkeuringen en meer. Het exacte oppervlak wordt bepaald door
de TypeBox-schema's die opnieuw worden geëxporteerd vanuit `packages/gateway-protocol/src/schema.ts`.

## Gerelateerd

- [Een Gateway-client bouwen](https://docs.openclaw.ai/gateway/clients)
- [OpenClaw insluiten](https://docs.openclaw.ai/gateway/embedding)
- [Bridge-protocol](/nl/gateway/bridge-protocol)
- [Gateway-draaiboek](/nl/gateway)
