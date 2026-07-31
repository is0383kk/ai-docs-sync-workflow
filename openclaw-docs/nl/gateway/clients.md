---
read_when:
    - Een operator, dashboard of WebChat-client bouwen buiten de OpenClaw-repository
    - Gateway-herverbinding, geschiedenis, goedkeuringen of apparaatkoppeling implementeren
    - Een client van derden bijwerken voor een nieuwe wire-versie van de Gateway
summary: Bouw een externe operator- of WebChat-client voor het Gateway WebSocket-protocol
title: Een Gateway-client bouwen
x-i18n:
    generated_at: "2026-07-27T05:33:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fa24b196ff1fa28fb3b64d49ac25597f22cf1945aea56029e78e4375f1bdddb7
    source_path: gateway/clients.md
    workflow: 16
---

Gebruik de gepubliceerde Gateway-pakketten om dashboards voor operators, WebChat-clients
en andere toepassingen van derden te bouwen. Deze handleiding behandelt de levenscyclus van de client rond
het wire-contract: authenticatie, mogelijkheden, herstel na opnieuw verbinden, geschiedenis,
abonnementen en versie-upgrades.

Lees voor framevormen, de handshake, fouten en het volledige methodeoppervlak de
[Gateway-protocolspecificatie](https://docs.openclaw.ai/gateway/protocol).

## Installeer de pakketten

```bash
npm install @openclaw/gateway-client @openclaw/gateway-protocol
```

<Note>
Deze pakketten worden geleverd met OpenClaw-releasereeksen. Tijdens de eerste uitrol kan npm
`E404` retourneren totdat de eerste OpenClaw-release met deze pakketten is gepubliceerd;
installeer ze pas nadat de onderstaande registerpagina's beschikbaar zijn.
</Note>

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  biedt schema's, runtimevalidators, TypeScript-typen, registers voor clientidentiteiten en
  mogelijkheden, lezers voor gestructureerde fouten en protocolversieconstanten.
  De npm-tarball bevat ook het gegenereerde
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  machineleesbare contract.
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  is de referentie-implementatie voor verbindingen. Importeer de pakketroot voor de Node-
  client en `@openclaw/gateway-client/browser` voor de browserveilige protocol-,
  apparaatauthenticatie- en herverbindingshelpers.

Het Node-toegangspunt beheert zijn WebSocket-transport. Een browserhost levert een WebSocket-
adapter plus persistente opslag en ondertekeningscallbacks voor de apparaatidentiteit en
het apparaattoken.

## Kies bereiken en koppel het apparaat

Een volledig interactieve chatclient die ook goedkeuringsprompts weergeeft, moet
`role: "operator"` aanvragen met deze bereiken:

| Bereik               | Gebruik het voor                                                                          |
| -------------------- | ----------------------------------------------------------------------------------------- |
| `operator.read`      | `chat.history`, `sessions.list`, `sessions.subscribe`, modelstatus en alleen-lezengebeurtenissen |
| `operator.write`     | `chat.send` en gewone sessiewijzigingen                                                   |
| `operator.approvals` | Het weergeven, tonen en afhandelen van exec- of plugingoedkeuringen                        |

Voeg `operator.questions` alleen toe als de client interactieve vragen afhandelt,
`operator.pairing` alleen als deze gekoppelde apparaten of nodes beheert, en
`operator.admin` alleen voor administratieve bewerkingen zoals `config.patch`.
De [referentie voor operatorbereiken](https://docs.openclaw.ai/gateway/operator-scopes)
definieert de volledige regels voor methoden en het moment van goedkeuring.

Maak niet handmatig een bearertoken per client door `openclaw.json` te bewerken. Configureer
de gedeelde bootstrap-authenticatie van de Gateway met `openclaw configure --section
gateway` of de `openclaw onboard --gateway-auth ...`-opties en laat vervolgens door
apparaatkoppeling het clienttoken aanmaken:

1. Bewaar een Ed25519-apparaatidentiteit permanent in de client.
2. Wacht op `connect.challenge`, onderteken de aan de challenge gebonden apparaatpayload en stuur
   `connect` met de aangevraagde operatorrol, bereiken en het gedeelde Gateway-token
   of wachtwoord voor bootstrap-authenticatie.
3. Als de Gateway gestructureerde `PAIRING_REQUIRED`-details retourneert, toon je de aanvraag-
   ID en pauzeer je of probeer je het opnieuw volgens `error.details.recommendedNextStep`.
4. Controleer de aanvraag op de Gateway-host met `openclaw devices list` en
   keur vervolgens precies die huidige aanvraag goed met `openclaw devices approve <requestId>`.
5. Maak opnieuw verbinding en bewaar `hello-ok.auth.deviceToken` permanent met de overeengekomen rol en
   bereiken. Gebruik dat apparaattoken voor latere verbindingen.

Upgrades van bereik of rol maken een nieuwe wachtende koppelingsaanvraag. Tokenrotatie kan
het goedgekeurde koppelingscontract niet uitbreiden. Zie de
[CLI voor apparaten](https://docs.openclaw.ai/cli/devices) voor opdrachten voor goedkeuring, rotatie en
intrekking.

## Maak clientmogelijkheden bekend

`connect.params.caps` beschrijft optioneel gedrag dat de client kan gebruiken. Dit
verleent geen autorisatie. Importeer namen uit `GATEWAY_CLIENT_CAPS` in plaats van
tekenreeksliteralen te dupliceren:

```ts
import { GATEWAY_CLIENT_CAPS } from "@openclaw/gateway-protocol/client-info";

const caps = [GATEWAY_CLIENT_CAPS.TOOL_EVENTS];
```

Het huidige register bevat `approvals`, `exec-approvals`, `inline-widgets`,
`run-tool-bindings`, `session-scoped-events`, `plugin-approvals`,
`task-suggestions`, `terminal-offset-seq`, `tool-events` en `ui-commands`.
Maak alleen mogelijkheden bekend die de client daadwerkelijk implementeert.

<Warning>
`tool-events` beheert live streaming van tooluitvoering. De Gateway registreert alleen
verbindingen die deze mogelijkheid bekendmaken als ontvangers van de gestructureerde
toolgebeurtenissen van een uitvoering. Zonder deze mogelijkheid ontvangt de verbinding geen live
toolgebeurtenissen en meldt de handshake geen fout.
</Warning>

Door mogelijkheden afgeschermde agenttools vormen een afzonderlijk gebruik van dezelfde declaratie. Als een
agenttool een clientmogelijkheid vereist, laat de Gateway die tool weg tenzij de
client van oorsprong elke vereiste mogelijkheid heeft bekendgemaakt.

## Herstel de status na opnieuw verbinden

Behandel elke geslaagde herverbinding als een nieuwe projectie over duurzame geschiedenis en
de huidige uitvoeringsstatus in het geheugen:

1. Herstel `sessions.subscribe` en het `sessions.messages.subscribe`-abonnement
   van de geselecteerde sessie.
2. Roep `chat.history` aan voor de geselecteerde `sessionKey` en vervang lokaal opgeslagen
   rijen door de geretourneerde `messages`-projectie.
3. Als `inFlightRun` aanwezig is, neem je de `runId`, gebufferde `text` en optionele
   `plan` ervan over. Neem de uitvoering ook over wanneer `text` leeg is.
4. Lees `sessionInfo.hasActiveRun` en `sessionInfo.activeRunIds`. Geef de voorkeur aan exact
   lidmaatschap in `activeRunIds` wanneer je bepaalt of een behouden uitvoering nog eigenaar is van
   de streaminginterface. Een ware `hasActiveRun` zonder vermelde ID kan een andere
   actieve runtimeprojectie vertegenwoordigen.
5. Stem volgende `agent`-gebeurtenissen af op `payload.runId` en `payload.seq`.
   Bewaar de hoogste geaccepteerde reeks onafhankelijk voor elke uitvoering, negeer een
   al geziene of lagere reeks en behandel een voorwaarts gat als reden om de
   gezaghebbende geschiedenis opnieuw te laden.

Het buitenste gebeurtenisframe heeft ook een optionele `seq`, die gebeurtenissen op de
huidige WebSocket-verbinding ordent. Deze wordt bij een nieuwe verbinding opnieuw ingesteld. De `seq` in
de payload van een `agent`-gebeurtenis wordt per uitvoering toegewezen en ordent de levenscyclus-,
assistent-, plan-, tool- en andere streamgebeurtenissen van die uitvoering.

## Gebruik geschiedenismetadata en stabiele ankers

Rijen die door `chat.history` worden geretourneerd, kunnen een `__openclaw`-metadata-envelop bevatten:

- `id` is de identiteit van de transcriptvermelding. Gebruik deze voor verankerde geschiedenisaanvragen,
  maar niet als unieke sleutel voor weergaverijen.
- `seq` is de positieve reeks van de transcriptrecord. Eén opgeslagen record kan
  naar meer dan één weergaverij worden geprojecteerd, dus houd verwante rijen met dezelfde `id` en reeks
  bij elkaar.
- `kind` identificeert synthetische rijen. Een Compaction-grens gebruikt
  `kind: "compaction"` en kan `tokensBefore` en `tokensAfter` bevatten wanneer een
  overeenkomend controlepunt die metingen heeft vastgelegd.

Blader achteruit met de waarden `hasMore` en `nextOffset` uit het antwoord. Numerieke
offsets beschrijven de huidige transcriptprojectie, dus bewaar ze niet als
langdurige bladwijzers over een reset of Compaction heen. Bewaar in plaats daarvan `__openclaw.id`.
Om rond een bekende rij te herstellen, roep je `chat.history` aan met `messageId` en de
`sessionId` die deze retourneerde. De Gateway kan dat anker oplossen vanuit de gearchiveerde
geschiedenis na een reset; verankerde antwoorden laten numerieke pagineringsmetadata bewust weg.

## Abonneer je in plaats van gebruik te pollen

Laad de oorspronkelijke catalogus met `sessions.list` en roep vervolgens `sessions.subscribe` eenmaal
per verbinding aan. Voeg `sessions.changed`-gebeurtenissen samen op basis van `sessionKey`. Payloads voor sessiewijzigingen
kunnen live `inputTokens`, `outputTokens`, `totalTokens`,
`totalTokensFresh`, `contextTokens`, `estimatedCostUsd`, instellingen voor antwoordgebruik
en de status van actieve uitvoeringen bevatten.

Sommige wijzigingsmeldingen zijn alleen invalidatiesignalen. Als bij een gebeurtenis de
rijvelden ontbreken die je weergave nodig heeft, vernieuw dan `sessions.list`. Poll `usage.cost` of
`sessions.usage` niet om een live sessielijst actueel te houden; reserveer die methoden voor
geaggregeerde of gedetailleerde rapporten op aanvraag.

## Vul exec-goedkeuringen aan

Een client met `operator.approvals` moet de gebeurtenislistener installeren zodra
`hello-ok` is voltooid en vervolgens `exec.approval.list` aanroepen om aanvragen aan te vullen die
van vóór de verbinding dateren. Stem de lijst en live
`exec.approval.requested`- / `exec.approval.resolved`-gebeurtenissen af op goedkeurings-ID, zodat een
overgang die tegelijk met de lijstaanvraag plaatsvindt niet verloren gaat of opnieuw tot leven wordt gewekt.

## Houd protocolversies bij

De huidige wire-versie is `4`. Algemene operator- en WebChat-clients moeten
de exacte huidige versie overeenkomen met `minProtocol: 4` en `maxProtocol: 4`.
Alleen geauthenticeerde nodeclients en lichtgewicht probes hebben het N-1-acceptatievenster,
momenteel protocol `3` tot en met `4`.

Protocolwijzigingen zijn eerst additief. `protocol.schema.json` bevat `since`-
metadata over de releasegeneratie en metadata over vereiste bereiken voor kernmethoden, maar een verhoging van de wire-
versie blijft een expliciete brekende gebeurtenis voor clients van derden. Zet de
geteste pakketversies vast, upgrade de client en Gateway samen wanneer de wire-
versie verandert en raadpleeg vóór elke upgrade het
[OpenClaw-wijzigingslogboek](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md).

## Gerelateerd

- [Gateway-protocol](https://docs.openclaw.ai/gateway/protocol)
- [OpenClaw insluiten](https://docs.openclaw.ai/gateway/embedding)
- [Gateway RPC-referentie](https://docs.openclaw.ai/reference/rpc)
- [Gateway-integraties voor externe apps](https://docs.openclaw.ai/gateway/external-apps)
