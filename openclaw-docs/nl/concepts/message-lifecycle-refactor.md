---
read_when:
    - Verzend- of ontvangstgedrag van kanalen refactoren
    - Wijzigingen aan inkomende kanaalberichten, antwoordverzending, uitgaande wachtrij, previewstreaming of bericht-API's van de Plugin-SDK
    - Een nieuwe kanaalplugin ontwerpen die duurzame verzendingen, ontvangstbevestigingen, voorbeelden, bewerkingen of nieuwe pogingen nodig heeft
summary: 'Status van de duurzame levenscyclus voor het ontvangen/verzenden van berichten: wat is uitgebracht, wat is gewijzigd ten opzichte van het oorspronkelijke ontwerp en wat nog openstaat'
title: Refactor van de levenscyclus van berichten
x-i18n:
    generated_at: "2026-07-27T06:12:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d21eda70b8be0de78677f4ff6d7547317112731d9e86a5bef58eac0268899818
    source_path: concepts/message-lifecycle-refactor.md
    workflow: 16
---

<Note>
Deze pagina is ontstaan als een toekomstgericht ontwerpvoorstel. De kern van dat
ontwerp is inmiddels uitgebracht in `src/channels/message/*` en de openbare
subpaden `openclaw/plugin-sdk/channel-outbound` / `channel-inbound`. Gebruik voor de
huidige API [API voor uitgaande kanalen](/nl/plugins/sdk-channel-outbound) en
[API voor inkomende kanalen](/nl/plugins/sdk-channel-inbound). Deze pagina houdt bij wat
is uitgebracht, waar de implementatie afwijkt van de oorspronkelijke schets en wat
nog openstaat.
</Note>

## Waarom deze refactor plaatsvond

De kanaalstack groeide uit verschillende lokale oplossingen: afzonderlijke helpers voor inkomend verkeer per
volwassenheidsniveau (`runtime.channel.inbound.run` voor eenvoudige adapters,
`runtime.channel.inbound.runPreparedReply` voor uitgebreide adapters), verouderde helpers voor antwoorddispatch
(`dispatchInboundReplyWithBase`, `recordInboundSessionAndDispatchReply`),
kanaalspecifieke previewstreaming en duurzaamheid van de uiteindelijke aflevering die achteraf werd toegevoegd aan
bestaande paden voor antwoordpayloads. Die vorm leidde tot te veel openbare concepten en
te veel plekken waar de afleveringssemantiek uiteen kon gaan lopen.

De betrouwbaarheidstekortkoming die het herontwerp afdwong:

```text
Telegram-pollingupdate bevestigd
  -> definitieve tekst van de assistent bestaat
  -> proces wordt opnieuw gestart voordat sendMessage slaagt
  -> definitief antwoord gaat verloren
```

Beoogde invariant: zodra de kern bepaalt dat er een zichtbaar uitgaand bericht moet bestaan,
moet de verzendintentie duurzaam zijn voordat de platformaanroep wordt geprobeerd, en moet het
platformontvangstbewijs na succes worden vastgelegd. Dat biedt standaard herstel met
ten minste één aflevering. Gedrag met exact één aflevering bestaat alleen waar een adapter
native idempotentie aantoont of een poging waarvan de uitkomst na verzending onbekend is, vergelijkt met
de platformstatus voordat deze opnieuw wordt uitgevoerd.

## Wat is uitgebracht

Het interne domein bevindt zich in `src/channels/message/*`:

| Bestand                     | Verantwoordelijk voor                                                                                                |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `types.ts`                  | Typecontracten voor adapter, verzendcontext, ontvangstbewijs en duurzame intentie                                    |
| `send.ts`                   | `withDurableMessageSendContext` / `sendDurableMessageBatch` — de duurzame verzendcontext                             |
| `receive.ts`                | `createMessageReceiveContext` — toestandsmachine voor het bevestigingsbeleid van inkomend verkeer                    |
| `live.ts`                   | Status van live preview en logica voor ter plaatse voltooien of terugvallen                                           |
| `state.ts`                  | `classifyDurableSendRecoveryState` — herstelclassificatie na onderbreking                                             |
| `receipt.ts`                | Normaliseert verzendresultaten van het platform naar `MessageReceipt`                                                |
| `capabilities.ts`           | Leidt vereiste mogelijkheden voor duurzame eindaflevering af uit een payload                                          |
| `contracts.ts`              | Verificatie van contractbewijs voor gedeclareerde adaptermogelijkheden                                                |
| `adapter.ts`                | `defineChannelMessageAdapter`                                                                                      |
| `outbound-bridge.ts`        | `createChannelMessageAdapterFromOutbound` — omhult verouderde functies `sendText`/`sendMedia`/`sendPayload`/`sendPoll` |
| `ingress-queue.ts`          | `createChannelIngressQueue` — duurzame wachtrij voor inkomende gebeurtenissen                                     |
| `durable-receive.ts`        | `createDurableInboundReceiveJournal` — journaal voor accepteren/in behandeling/voltooien/vrijgeven voor deduplicatie van inkomend verkeer |
| `inbound-reply-dispatch.ts` | `dispatchChannelInboundReply` en wrappers met verouderde namen                                                      |
| `reply-pipeline.ts`         | `createChannelReplyPipeline`, helpers voor antwoordprefix en typcallback                                           |

Openbaar oppervlak: `openclaw/plugin-sdk/channel-outbound` (helpers voor verzending/ontvangstbewijzen/duurzaamheid/liveweergave/antwoordpijplijn)
en `openclaw/plugin-sdk/channel-inbound` (context voor inkomend verkeer, `runChannelInboundEvent`,
`dispatchChannelInboundReply`). Zie die pagina's voor adaptervoorbeelden, huidige
typenamen en migratieopmerkingen — zij zijn de gezaghebbende bron voor de API-
vorm, niet de onderstaande schetsen.

### Verzendcontext

`withDurableMessageSendContext` biedt kanaalcode de stappen `render`, `previewUpdate`,
`send`, `edit`, `delete`, `commit` en `fail` rond één uitgaand
bericht. `sendDurableMessageBatch` is de wrapper voor het gebruikelijke geval: renderen, verzenden
en vervolgens vastleggen bij `sent`/`suppressed` of mislukken bij een fout.

`sendDurableMessageBatch` retourneert één gediscrimineerd resultaat:

| Status           | Betekenis                                                                         |
| ---------------- | --------------------------------------------------------------------------------- |
| `sent`           | Ten minste één zichtbaar platformbericht is afgeleverd                            |
| `suppressed`     | Geen platformbericht moet als ontbrekend worden beschouwd (door hook geannuleerd, proefuitvoering enz.) |
| `partial_failed` | Ten minste één bericht is afgeleverd voordat een latere payload of bijwerking mislukte |
| `failed`         | Er is geen platformontvangstbewijs geproduceerd                                   |

Duurzaamheid is `required`, `best_effort` of `disabled`
(`MessageDurabilityPolicy` in `src/channels/message/types.ts`). `required`
stopt veilig wanneer de duurzame intentie niet kan worden geschreven; `best_effort` valt
terug op directe verzending wanneer persistentie niet beschikbaar is; `disabled` behoudt het
gedrag van directe verzending van vóór de refactor. Verouderde compatibiliteitshelpers gebruiken standaard
`disabled` en leiden `required` niet af alleen omdat een kanaal een generieke
adapter voor uitgaand verkeer heeft.

De grens die gevaarlijk blijft: nadat de platformaanroep slaagt en voordat
het ontvangstbewijs wordt vastgelegd. Als het proces daar stopt, kan de kern niet weten of het
platformbericht bestaat, tenzij de adapter `reconcileUnknownSend` declareert.
Die hook classificeert een onderbroken verzending als `sent`, `not_sent` of
`unresolved`; alleen `not_sent` staat herhaling toe. Kanalen zonder reconciliatie
vallen terug op de status `unknown_after_send` (`src/channels/message/state.ts`,
`src/infra/outbound/delivery-queue-recovery.ts`) en mogen alleen kiezen voor herhaling met
ten minste één aflevering als dubbele zichtbare berichten een aanvaardbare, gedocumenteerde
afweging voor dat kanaal zijn.

### Ontvangstcontext

`createMessageReceiveContext` houdt de bevestigings-/afwijzingsstatus per inkomende gebeurtenis bij met een
idempotente `ack()` en expliciete `nack(error)`. Het bevestigingsbeleid
(`ChannelMessageReceiveAckPolicy`) is een van:

| Beleid                 | Bevestigt wanneer                                                                                |
| ---------------------- | ------------------------------------------------------------------------------------------------ |
| `after_receive_record` | De kern voldoende metagegevens van inkomend verkeer heeft opgeslagen om een heraflevering te dedupliceren/routeren |
| `after_agent_dispatch` | De agentuitvoering is verzonden                                                                  |
| `after_durable_send`   | De duurzame uitgaande verzending voor deze beurt is vastgelegd                                   |
| `manual`               | De aanroeper het bevestigingstijdstip expliciet beheert (standaard voor adapters die geen beleid declareren) |

Telegram-polling gebruikt dit om een watermerk voor veilig voltooide updates op te slaan
(`safeCompletedUpdateId` in `extensions/telegram/src/bot-update-tracker.ts`):
grammY neemt nog steeds elke update waar wanneer deze de middlewareketen binnenkomt, maar
OpenClaw verhoogt het opgeslagen herstartwatermerk alleen tot voorbij updates die
de dispatch hebben voltooid, zodat mislukte of nog in behandeling zijnde updates na een herstart opnieuw worden uitgevoerd.
De bovenliggende `getUpdates`-offset van Telegram blijft eigendom van grammY; een volledig
duurzame pollingbron die heraflevering op platformniveau voorbij dit
watermerk beheert, is niet gebouwd (zie Openstaande vragen).

### Live preview

`src/channels/message/live.ts` modelleert preview/bewerken/voltooien als één levenscyclus:
`createLiveMessageState`, `markLiveMessagePreviewUpdated`,
`markLiveMessageFinalized`, `markLiveMessageCancelled` en
`deliverFinalizableLivePreviewAdapter` (een definitieve bewerking opbouwen vanuit een concept, deze
toepassen en terugvallen op een normale verzending wanneer bewerken niet mogelijk is of mislukt).
`LiveMessageState.phase` is `idle | previewing | finalizing | finalized |
cancelled`; `canFinalizeInPlace` bepaalt of een preview via een bewerking het definitieve
bericht kan worden in plaats van via een nieuwe verzending.

### Duurzame ontvangstbewijzen

`MessageReceipt` (`src/channels/message/types.ts`) normaliseert een of meer
platformbericht-id's van één logische verzending naar `platformMessageIds` plus
`parts` per onderdeel (soort, index, thread-id, antwoord-op-id). Een primair id wordt bewaard
voor threads en latere bewerkingen. Hierdoor kunnen leveringen met meerdere onderdelen (tekst
plus media, opgeknipte tekst, terugval naar kaart) na een herstart opnieuw worden uitgevoerd en
gededupliceerd.

### Verkleining van de openbare SDK

De refactor heeft het volgende opgenomen of afgeschaft: `reply-runtime`, `reply-dispatch-runtime`,
`reply-reference`, `reply-chunking`, `reply-payload`-helpers die als openbare
API waren beschikbaar gesteld, `inbound-reply-dispatch`, `channel-reply-pipeline` en de meeste openbare toepassingen
van de oude facade voor uitgaand verkeer. `src/plugin-sdk/channel-message.ts` is nu een
`@deprecated`-barrel voor herexport die verwijst naar `channel-outbound` /
`channel-inbound`; runtime-aliassen van `channel.turn` zijn verwijderd en de oude
documentatiepagina `/plugins/sdk-channel-turn` verwijst door naar
[API voor inkomende kanalen](/nl/plugins/sdk-channel-inbound). Nieuwe plugincode moet zich
rechtstreeks richten op `channel-outbound` en `channel-inbound`.

## Waar de implementatie afweek van het oorspronkelijke ontwerp

De onderstaande ontwerpschets is nooit letterlijk zoals beschreven uitgebracht. Deze is bewaard voor
historische nauwkeurigheid; beschouw deze typenamen niet als de huidige API.

- **Geen `MessageOrigin` / `shouldDropOpenClawEcho`.** Het oorspronkelijke plan voorzag
  in een `source: "openclaw"`-oorsprongstag op berichten over Gateway-fouten, plus een
  gedeeld predicaat dat getagde, door bots geschreven echo's in gedeelde ruimtes verwijdert
  vóór `allowBots`-autorisatie. Dat type en predicaat bestaan niet in
  de codebase. `allowBots` zelf is een echte configuratiesleutel per kanaal (Slack,
  Discord, Google Chat en andere), maar het mechanisme voor oorsprongstags dat deze
  moest beschermen, is nooit gebouwd. Onderdrukking van echo's van Gateway-fouten in
  ruimtes met bots blijft een open tekortkoming, geen uitgebrachte garantie.
- **Geen uniforme `core.messages.receive/send/live/state`-naamruimte.** De
  uitgebrachte functies bevinden zich rechtstreeks in `src/channels/message/*`
  (`withDurableMessageSendContext`, `createMessageReceiveContext`,
  `createLiveMessageState`, `classifyDurableSendRecoveryState`) in plaats van
  achter een `core.messages.*`-facade.
- **Geen generiek genormaliseerd berichttype `ChannelMessage` / `MessageTarget` / `MessageRelation`.**
  De kern geeft nog steeds concrete antwoordpayloads
  (`ReplyPayload`) en kanaalspecifieke contexten door aan de verzendadapters,
  in plaats van één platformneutrale berichtvorm met een `kind: "reply" |
"followup" | "broadcast" | "system"`-relatie.
- **Namen van bevestigingsbeleid verschillen van de schets.** Uitgebracht:
  `after_receive_record | after_agent_dispatch | after_durable_send | manual`.
  De oorspronkelijke schets gebruikte `immediate | after-record | after-durable-send |
manual` met een redenveld voor webhooktime-outs; die vorm is niet gebouwd.
- **Mogelijkheidssleutels van `DurableFinalDeliveryRequirementMap` vervingen het geschetste
  `MessageCapabilities`-object.** Mogelijkheden zijn platte booleaanse vlaggen (`text`,
  `media`, `poll`, `payload`, `silent`, `replyTo`, `thread`, `nativeQuote`,
  `messageSendingHooks`, `batch`, `reconcileUnknownSend`, `afterSendSuccess`,
  `afterCommit`) die via `verifyDurableFinalCapabilityProofs` worden geverifieerd in plaats
  van een geneste structuur in de stijl van `text.chunking` / `attachments.voice`.

## Concrete migratierisico's (nog steeds relevant)

Deze kanaalspecifieke bijwerkingen dateren van vóór de refactor en moeten blijven
werken via de nieuwe verzendpaden. Ze zijn niet hypothetisch: ze zijn stuk voor stuk
geïmplementeerd en essentieel voor de werking van vandaag.

- **iMessage** (`extensions/imessage/src/monitor/echo-cache.ts`,
  `persisted-echo-cache.ts`): de monitor registreert verzonden berichten na een geslaagde verzending in een echo-
  cache. Duurzame definitieve verzendingen moeten die cache nog steeds vullen,
  anders kan OpenClaw zijn eigen antwoorden opnieuw verwerken als inkomende gebruikersberichten.
- **Tlon** (`extensions/tlon/src/monitor/index.ts`): voegt een optionele modelhandtekening toe
  en registreert threads waaraan is deelgenomen na antwoorden in groepen. Duurzame
  aflevering mag die effecten niet omzeilen.
- **Discord en andere voorbereide dispatchers** beheren directe aflevering en
  voorbeeldgedrag al zelf. Een kanaal is niet end-to-end duurzaam totdat de voorbereide
  dispatcher definitieve berichten expliciet via de verzendcontext routeert; ga niet uit
  van dekking door alleen de generieke adapter.
- **Stille fallback-aflevering van Telegram** moet na chunking/fallback-
  projectie de volledige array met geprojecteerde payloads afleveren, niet alleen de eerste payload.
- **LINE, Zalo, Nostr** en vergelijkbare hulppaden kunnen verwerking van
  antwoordtokens, mediaproxying, caches voor verzonden berichten of uitsluitend voor callbacks
  bestemde doelen hebben. Ze blijven kanaalgestuurde aflevering gebruiken totdat die semantiek door
  de verzendadapter wordt gerepresenteerd en door tests wordt gedekt.
- **Helpers voor directe DM's** kunnen een antwoordcallback hebben die het enige juiste
  transportdoel is. Generieke uitgaande verzending mag niet op basis van onbewerkte
  platformvelden een doel raden en die callback overslaan.

## Classificatie van fouten

Adapters classificeren transportfouten in gesloten categorieën in `DeliveryFailureKind`-stijl
(tijdelijk, snelheidslimiet, authenticatie, toestemming, niet gevonden, ongeldige
payload, conflict, geannuleerd, onbekend). Kernbeleid:

- Probeer tijdelijke fouten en fouten door snelheidslimieten opnieuw.
- Probeer fouten door een ongeldige payload niet opnieuw, tenzij er een renderfallback bestaat.
- Probeer authenticatie- of toestemmingsfouten niet opnieuw totdat de configuratie is gewijzigd.
- Laat bij niet gevonden de live finalisatie terugvallen van bewerken naar een nieuwe verzending wanneer
  het kanaal aangeeft dat dit veilig is.
- Gebruik bij een conflict ontvangstbewijs-/idempotentiestatus om te bepalen of het bericht
  al bestaat.
- Elke fout nadat de platformaanroep mogelijk is geslaagd maar voordat het ontvangstbewijs
  is vastgelegd, wordt `unknown_after_send`, tenzij de adapter bewijst dat de platformbewerking
  niet heeft plaatsgevonden.

## Openstaande vragen

- Of Telegram uiteindelijk de grammY-pollingrunner (`1.43.0`) moet vervangen
  door een volledig duurzame pollingbron die herlevering op platformniveau beheert,
  en niet alleen OpenClaws persistente herstartwatermerk
  (`safeCompletedUpdateId`).
- Of de status van livevoorbeelden in dezelfde record als de intentie voor de definitieve
  verzending moet staan, of in een naastgelegen opslag voor livestatus.
- Of echo-onderdrukking bij Gateway-fouten in gedeelde ruimten met bots
  het oorspronkelijk geplande mechanisme voor oorsprongstags nodig heeft, een eenvoudiger
  contract per kanaal, of buiten het bereik valt.
- Welke kanalen native ondersteuning voor oorsprong/metadata hebben voor echo-
  onderdrukking tussen bots, en welke een persistente uitgaande registratie nodig hebben.

## Gerelateerd

- [Berichten](/nl/concepts/messages)
- [Streaming en opdelen](/nl/concepts/streaming)
- [Voortgangsconcepten](/nl/concepts/progress-drafts)
- [Beleid voor opnieuw proberen](/nl/concepts/retry)
- [Uitgaande API voor kanalen](/nl/plugins/sdk-channel-outbound)
- [Inkomende API voor kanalen](/nl/plugins/sdk-channel-inbound)
