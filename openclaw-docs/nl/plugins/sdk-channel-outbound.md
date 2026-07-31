---
read_when:
    - Je bouwt of refactort het verzendpad van een Plugin voor een berichtenkanaal
    - Je hebt duurzaam afleveren van definitieve antwoorden, ontvangstbevestigingen, afronding van livevoorbeelden of een beleid voor ontvangstbevestiging nodig
    - Je migreert van helpers voor kanaalberichten of verouderde antwoordverzending
summary: 'API voor de levenscyclus van uitgaande berichten voor kanaalplugins: adapters, ontvangstbewijzen, duurzame verzendingen, livevoorbeelden en helpers voor de antwoordpijplijn'
title: API voor uitgaande kanaalberichten
x-i18n:
    generated_at: "2026-07-27T05:43:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8edeca81d2e9261f33be1d538153caaea87caedb90dfccac33dd227c924501f1
    source_path: plugins/sdk-channel-outbound.md
    workflow: 16
---

Channelplugins maken het gedrag voor uitgaande berichten beschikbaar vanuit
`openclaw/plugin-sdk/channel-outbound`. Gebruik
`openclaw/plugin-sdk/channel-inbound` voor de orkestratie van ontvangst/context/dispatch.

Core beheert wachtrijvorming, duurzaamheid, de duurzame **ingangsmonitor en verwerking**
(`createChannelIngressMonitor`, `createChannelIngressDrain` en
`openChannelIngressDrain`), generiek beleid voor nieuwe pogingen, de levenscyclus voor overname van beurten
(`turnAdoptionLifecycle` / `bindIngressLifecycleToReplyOptions`), hooks,
ontvangstbewijzen en de gedeelde tool `message`. De plugin beheert native
aanroepen voor verzenden/bewerken/verwijderen, normalisatie van doelen, platformspecifieke threads, geselecteerde
citaten, meldingsvlaggen, accountstatus, inspectie van binnenkomende gegevens en codering van payloads,
baansleutels, predicaten voor niet-opnieuw-uitvoerbare fouten, optionele autorisatie voor vervanging
en platformspecifieke neveneffecten.

## Duurzame ingangsmonitors

Gebruik `createChannelIngressMonitor(...)` wanneer een kanaal geaccepteerde
transportgebeurtenissen vóór dispatch duurzaam moet opslaan. Dit combineert een wachtrij en verwerking voor kanaalingang
met de gedeelde levenscyclus voor toelating, polling, opschoning, levering en afsluiting.
Gebruik het onderliggende `createChannelIngressDrain(...)` alleen wanneer het transport
een wezenlijk ander contract voor toelating of verwerking beheert.

De vereiste opties zijn:

| Optie                            | Contract                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `queue`                          | Een `ChannelIngressQueue`, of een luie factory die de accountgebonden wachtrij opent.                                                                                                                                                                                                                                  |
| `inspect(raw, context)`          | Retourneert de stabiele `eventId` en geserialiseerde `laneKey`, of `null` voor een genegeerde gebeurtenis. Feiten op het moment van claimen moeten overeenkomen met de opgeslagen id en baan.                                                                                                                                                                    |
| `payload`                        | Levert de payloadversie plus serialisatie/deserialisatie van de inhoud. Gebruik `storage: "raw-event"` voor de standaard tekenreeksenvelop `{ version, rawEvent }`, of geef aangepaste callbacks voor coderen/decoderen op voor een bestaande kanaalspecifieke vorm. `createClaimError` classificeert ongeldige versies of een gewijzigde identiteit. |
| `deliver(raw, lifecycle, claim)` | Verstuurt één gedecodeerde gebeurtenis en ontvangt de volledige overnamelevenscyclus. Kan `completed`, `deferred`, `failed-retryable` of niets retourneren.                                                                                                                                                                |
| `pollIntervalMs`                 | Plant herstel-/verwerkingspolls zolang de monitor actief is.                                                                                                                                                                                                                                                     |
| `retention`                      | Levert het opschoningsinterval en de TTL's en limieten voor voltooide/mislukte vermeldingen.                                                                                                                                                                                                                                              |

De monitor serialiseert toelatingen, zodat back-off bij toevoegen de volgorde binnen een baan niet kan omkeren. De
standaard begrensde vertragingen voor toevoegen zijn `0`, `100` en `300` ms; bij uitputting wordt
de transportcallback geweigerd in plaats van een gebeurtenis te dispatchen die niet
duurzaam is opgeslagen. Op het moment van claimen decodeert de monitor de payload met versie, voert `inspect` opnieuw uit en
weigert deze een niet-overeenkomende id of baan vóór levering.

`deliver` ontvangt `onAdopted`, `onDeferred`, `onAdoptionFinalizing`,
`onAbandoned` en `abortSignal`. Terugkeren zonder expliciete overdracht markeert een
terminale gebeurtenis zonder dispatch als overgenomen. `admission` is altijd `exclusive`. Een
uitgestelde overdracht houdt de claim vast, terwijl afsluiting of afbreking niet-overgenomen
werk opnieuw uitvoerbaar laat. De monitor volgt levering onafhankelijk van de afhandeling van claims,
omdat overname een rij als verwijderd kan markeren voordat de leveringsbelofte van het kanaal
terugkeert.

Optionele instellingen omvatten aangepaste vertragingen voor toevoegen, een optieblok `drain` voor
geavanceerde volgorde/concurrency/beleid voor nieuwe pogingen bij verwerking, een extern `abortSignal`, een
klok, rapportage van verwerkingsfouten, een factory voor fouten bij gestopte toestand en toelatingsbeleid.
De geretourneerde monitor maakt `admit`, `start`, `pause`, `stop`, `waitForIdle`,
`isRunning` en `isStopped` beschikbaar. `stop` handelt eerst geaccepteerde toelatingen af, breekt daarna
de verwerking af en verwijdert deze, wacht op de verwerking en actieve leveringen, en
verwijdert deze opnieuw om de racecondition bij luie creatie af te sluiten.

Houd transportspecifieke redactie, validatie van onbewerkte enveloppen, classificatie als
niet-opnieuw-uitvoerbaar en de opgeslagen payloadvorm in de plugin. Webhook-transporten
mogen pas bevestigen nadat `admit` is voltooid; transporten zonder herhalingsmogelijkheid moeten
uitputting bij duurzaam toevoegen melden in plaats van stilzwijgend te dispatchen.

## Adapter

De meeste plugins definiëren één `message`-adapter:

```ts
import {
  defineChannelMessageAdapter,
  createMessageReceiptFromOutboundResults,
} from "openclaw/plugin-sdk/channel-outbound";

export const demoMessageAdapter = defineChannelMessageAdapter({
  id: "demo",
  durableFinal: {
    capabilities: {
      text: true,
      replyTo: true,
      thread: true,
      messageSendingHooks: true,
    },
  },
  send: {
    text: async ({ cfg, to, text, accountId, replyToId, threadId, signal }) => {
      const sent = await sendDemoMessage({
        cfg,
        to,
        text,
        accountId: accountId ?? undefined,
        replyToId: replyToId ?? undefined,
        threadId: threadId == null ? undefined : String(threadId),
        signal,
      });

      return {
        receipt: createMessageReceiptFromOutboundResults({
          results: [{ channel: "demo", messageId: sent.id, conversationId: to }],
          kind: "text",
          threadId: threadId == null ? undefined : String(threadId),
          replyToId: replyToId ?? undefined,
        }),
      };
    },
  },
});
```

Declareer alleen mogelijkheden die het native transport daadwerkelijk behoudt. Dek
elke gedeclareerde mogelijkheid voor verzenden, ontvangstbewijzen, livevoorbeelden en ontvangstbevestigingen af met
de contracthelpers die vanuit dit subpad worden geëxporteerd.

## Onderdrukking van uitgaande echo's

Wanneer een platform het eigen uitgaande bericht van de plugin opnieuw als inkomend bericht kan afleveren, roep je `recordOutboundMessageIdentity(...)` aan met het kanaal, account, gesprek en een stabiele platformbericht- of bronidentiteit. Het gedeelde pad voor inkomende beurten verwijdert overeenkomende identiteiten gedurende een begrensd venster van 30 seconden vóór sessieregistratie of dispatch naar de agent; een bronidentiteit kan vóór verzending worden gereserveerd of worden vernieuwd wanneer een kanaalroute wordt verwijderd om leveringsraceconditions af te sluiten. `isRecentOutboundMessageIdentity(...)` maakt dezelfde query beschikbaar voor kanaaldiagnostiek en tests. Onderhoud geen parallelle kanaallokale TTL-cache voor dezelfde stabiele identiteit.

## Opschoning van platte tekst

Gebruik `sanitizeForPlainText(...)` wanneer een uitgaande adapter de
ondersteunde HTML-opmaaktags moet omzetten in lichtgewicht tekstopmaak. Standaard blijven
de bestaande chatmarkeringen voor vet en doorhalen behouden. Geef
`{ style: "markdown" }` alleen door wanneer het kanaal het resultaat opnieuw als Markdown verwerkt:

```ts
import { sanitizeForPlainText } from "openclaw/plugin-sdk/channel-outbound";

const chatText = sanitizeForPlainText(text);
const markdownText = sanitizeForPlainText(text, { style: "markdown" });
```

De Markdown-stijl gebruikt `**bold**` en `~~strikethrough~~`; cursief en inline
code behouden in beide stijlen `_italic_` en backtickmarkeringen. Selecteer de stijl bij
de kanaalgrens in plaats van markeringstekst na opschoning te herschrijven.

## Leveringsbewijs

Een `MessageReceipt` registreert het resultaat dat een kanaaladapter retourneert. Concrete
platformbericht-id's tonen aan dat het verzendpad van het platform het
bericht heeft geaccepteerd; ze bewijzen niet dat het apparaat van een ontvanger het heeft weergegeven of gelezen.
Ontvangstbewijzen zonder platformbericht-id's zijn uitsluitend lokale metagegevens voor ontvangstbewijzen.
Kanalen met leesbevestigingen of informatie over levering aan apparaten moeten deze feiten
via een afzonderlijk kanaalspecifiek pad bijhouden.

Als een kanaaladapter kan bewijzen dat het opnieuw proberen van een fout geen
voor de ontvanger zichtbare verzending kan dupliceren en er geen aanroep met finalisatiemogelijkheid is begonnen, genereer dan
`new PlatformMessageNotDispatchedError("...", { cause: error })` vanuit
`openclaw/plugin-sdk/error-runtime`. Core kan dan verouderd bewijs van verzendpogingen
wissen en de intentie in de wachtrij veilig opnieuw uitvoeren. Alleen de adapter die de
uiteindelijke dispatchgrens beheert, mag dit beweren. Gebruik de markering nooit nadat een
finalisatie-/verzendaanroep begint of een dubbelzinnig resultaat retourneert; een onjuiste markering kan
berichten dupliceren.

## Bestaande uitgaande adapters

Als het kanaal al een compatibele `outbound`-adapter heeft, leid je de
berichtadapter daarvan af in plaats van verzendcode te dupliceren:

```ts
import { createChannelMessageAdapterFromOutbound } from "openclaw/plugin-sdk/channel-outbound";

export const messageAdapter = createChannelMessageAdapterFromOutbound({
  id: "demo",
  outbound,
  durableFinal: {
    capabilities: {
      text: true,
      media: true,
    },
  },
});
```

## Duurzame verzendingen

Runtime-helpers voor verzending zijn ook beschikbaar op `channel-outbound`:

- `sendDurableMessageBatch(...)`
- `withDurableMessageSendContext(...)`
- `deliverInboundReplyWithMessageSendContext(...)`
- helpers voor conceptstreaming/voortgang, zoals `resolveChannelDraftStreamingChunking(...)`

`sendDurableMessageBatch(...)` retourneert één expliciete uitkomst:

| Uitkomst          | Betekenis                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------- |
| `sent`           | ten minste één zichtbaar platformbericht is door het verzendpad van het platform geaccepteerd            |
| `suppressed`     | geen enkel platformbericht mag als ontbrekend worden beschouwd                                        |
| `partial_failed` | ten minste één platformbericht is geaccepteerd voordat een latere payload of een later neveneffect mislukte |
| `failed`         | er is geen platformontvangstbewijs geproduceerd                                                        |

Gebruik `payloadOutcomes` wanneer een batch verzonden, onderdrukte en mislukte
payloads combineert. Leid annulering door hooks niet af uit een leeg verouderd
resultaat voor directe levering.

## Toelating voor uitgestelde levering

Gebruik `message.durableFinal.admitDeferredDelivery(...)` wanneer een herleid account
door Core beheerde uitgaande of uitgestelde levering niet veilig kan accepteren. Core roept
deze hook synchroon aan vóór actief uitgaand werk, inclusief paden die
duurzame opslag in de wachtrij overslaan, en opnieuw voordat een herstelde intentie wordt afgespeeld. De context
omvat `cfg`, `channel`, `to`, `accountId` en een `phase` van `live` of
`recovery`.

Retourneer `{ status: "allowed" }` om door te gaan. Retourneer
`{ status: "permanent_rejection", reason }` wanneer de levering niet
duurzaam mag worden opgeslagen, rechtstreeks mag worden verzonden of opnieuw mag worden afgespeeld. Een actieve weigering mislukt vóór het aanmaken van de wachtrij,
berichthooks of platformwerk. Een weigering tijdens herstel markeert de
wachtrijregistratie als mislukt en slaat reconciliatie en opnieuw afspelen over. Als de hook ontbreekt,
is dit toegestaan.

De hook is een synchrone toelatingsbeslissing, geen verzendpad. Lees alleen
reeds geladen configuratie of runtimestatus; voer geen netwerk-, bestandssysteem- of
andere asynchrone I/O uit. Contracttests moeten beide fasen en beide
resultaatvarianten doorlopen via `ChannelMessageDurableFinalAdapter` vanuit
`openclaw/plugin-sdk/channel-outbound`.

## Compatibiliteitsdispatch

Stel de dispatch voor inkomende antwoorden samen via `dispatchChannelInboundReply(...)`
vanuit `channel-inbound`. Houd platformlevering in de leveringsadapter; gebruik
`channel-outbound` voor berichtadapters, duurzame verzendingen, ontvangstbevestigingen, live
voorvertoning en opties voor de antwoordpijplijn.
