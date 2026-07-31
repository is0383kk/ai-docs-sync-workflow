---
read_when:
    - Je bouwt of herstructureert het ontvangstpad van een Plugin voor een berichtenkanaal
    - Je hebt gedeelde opbouw van inkomende context, sessieregistratie of voorbereide antwoordverzending nodig
    - Je migreert oude helpers voor kanaalbeurten naar inbound/message-API's
summary: 'Helpers voor inkomende gebeurtenissen voor kanaalplugins: contextopbouw, gedeelde runnerorkestratie, sessierecord en voorbereide antwoordverzending'
title: API voor inkomende kanaalberichten
x-i18n:
    generated_at: "2026-07-27T05:10:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 854408ca42cfe1e1b48e4fd223b176f438e1db28deb9a5aa33eea8238127d9df
    source_path: plugins/sdk-channel-inbound.md
    workflow: 16
---

Ontvangstpaden van kanalen volgen één flow:

```text
platformgebeurtenis -> inkomende feiten/context -> agentantwoord -> berichtbezorging
```

Gebruik `openclaw/plugin-sdk/channel-inbound` voor normalisatie van inkomende gebeurtenissen,
opmaak, roots en orkestratie. Gebruik
`openclaw/plugin-sdk/channel-outbound` voor systeemeigen verzending, ontvangstbewijs, duurzame
bezorging en livevoorbeeldgedrag.

## Kernhelpers

```ts
import {
  buildChannelInboundEventContext,
  runChannelInboundEvent,
  dispatchChannelInboundReply,
} from "openclaw/plugin-sdk/channel-inbound";
```

- `buildChannelInboundEventContext(...)`: projecteert genormaliseerde kanaalfeiten
  in de prompt-/sessiecontext. Geef door het kanaal beheerde metadata van afzender/chat
  door via `channelContext`, die Plugin-hooks zien als `ctx.channelContext`.
  Breid `PluginHookChannelSenderContext` of `PluginHookChannelChatContext`
  vanuit dit subpad uit met kanaalspecifieke velden.
- `runChannelInboundEvent(...)`: voert opname, classificatie, preflight, oplossing,
  registratie, verzending en afronding uit voor één inkomende platformgebeurtenis.
- `dispatchChannelInboundReply(...)`: registreert en verzendt een reeds
  samengesteld inkomend antwoord met een bezorgingsadapter.

Houd voor inkomende gebeurtenissen die alleen media bevatten de berichttekst en opdrachttekst leeg en
geef per systeemeigen bijlage één `ChannelInboundMediaInput`-feit door. Wanneer een omgevingsgebonden
geschiedenisregel of een andere uitsluitend tekstuele drager die feiten moet beschrijven, gebruik je
`formatMediaPlaceholderText(media)`. Deze classificeert elk feit op basis van `kind`, het MIME-
type en vervolgens de extensie van het pad of de URL; nog niet gedownloade systeemeigen bijlagen moeten
elk nog steeds één feit met alleen een type opleveren. Gebruik de formatter niet om de
primaire inkomende berichttekst te genereren.

Normaliseer door de Plugin beheerde bijlagerecords met `toInboundMediaFacts(...)` en
geef daarna de resulterende geordende array door via het veld `media` van de context:

```ts
const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

De arraypositie is de identiteit van de bijlage. `transcribed`, `messageId` en
`workspaceDir` per feit vervangen de verouderde parallelle index-/werkruimtevelden. De
contextvelden `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` en `MediaStaged`,
plus `buildChannelInboundMediaPayload(...)`, blijven alleen beschikbaar als verouderde
compatibiliteitsvoorziening. Nieuwe plugins mogen ze niet aanmaken of uitlezen.

Gebundelde/systeemeigen kanalen die het geïnjecteerde Plugin-runtimeobject al ontvangen,
kunnen dezelfde helpers aanroepen onder `runtime.channel.inbound.*` in plaats van
dit subpad rechtstreeks te importeren:

```ts
await runtime.channel.inbound.run({
  channel: "demo",
  accountId,
  raw: platformEvent,
  adapter: {
    ingest: normalizePlatformEvent,
    resolveTurn: resolveInboundReply,
  },
});
```

Stel invoer voor `dispatchChannelInboundReply(...)` samen voor compatibiliteitsdispatchers
die platformbezorging in de bezorgingsadapter houden. Nieuwe verzendpaden
moeten in plaats daarvan berichtadapters en duurzame berichthelpers uit
`channel-outbound` gebruiken.

## Contract voor afhandeling van bezorging

`ChannelInboundTurnPlan.delivery` beheert de systeemeigen verzending voor elke logische
antwoordpayload. Core beheert de volgorde van uitgaande hooks en, wanneer de adapter zich hiervoor aanmeldt,
de uiteindelijke observatie van `message_sent`. Houd deze verantwoordelijkheden gescheiden, zodat
één payload geen dubbele eindgebeurtenissen kan produceren.

De velden van het bezorgingsresultaat hebben de volgende betekenis:

| Veld                    | Contract                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                | Door de provider geaccepteerde zichtbare tekst voor de logische payload na systeemeigen opmaak of afronding. Laat dit weg om de voorbereide payloadtekst te gebruiken voor de uiteindelijke observatie. Verzendingen met alleen media kunnen dit weglaten.                             |
| `messageIds` / `receipt` | Werkelijke provideridentiteiten voor de zichtbare verzending. Geef de voorkeur aan een `MessageReceipt`; core gebruikt de primaire provider-id daarvan voor `message_sent`.                                                                                            |
| `visibleReplySent`       | Stel alleen in op `false` wanneer de provider geen zichtbaar voorbeeld of definitief bericht heeft geproduceerd. Core genereert voor dat resultaat geen geslaagde `message_sent`.                                                                          |
| `finalization`           | Een promise voor uitgestelde systeemeigen afhandeling van dezelfde logische payload, zoals het sluiten of bewerken van een ter plaatse streamende kaart. De opgeloste velden overschrijven het onmiddellijke resultaat vóór de uiteindelijke observatie en `onDelivered`. |

Stel de optie `observeMessageSent` van de bezorgingsadapter in op `true` wanneer core
de canonieke Plugin- en interne `message_sent`-gebeurtenissen moet genereren voor de
niet-duurzame verzendingen van deze adapter. Retourneer deze optie niet vanuit `deliver` en
genereer die gebeurtenissen ook niet in de Plugin. Duurzame verzendingen worden al gegenereerd via
de gedeelde eigenaar van uitgaande berichten en worden niet gedupliceerd.

Retourneer één resultaat per logische payload. `finalization` is geen tweede verzending en
mag `reply_payload_sending` of `message_sending` niet opnieuw uitvoeren. Zodra
`deliver` retourneert, observeert core de afwijzing van de afrondingspromise, zodat deze
niet onafgehandeld kan blijven; core wacht nog steeds op de oorspronkelijke promise nadat de verzending
van het antwoord is afgehandeld. Vervolgens genereert core maximaal één uiteindelijke observatie per payload
met de afgeronde inhoud en provider-id. `onDelivered`, indien aanwezig,
ontvangt het afgehandelde resultaat na die observatie.

Wijs `deliver` of `finalization` af wanneer systeemeigen bezorging mislukt. Als geen
providerverzending is geprobeerd, werp je `PlatformMessageNotDispatchedError` vanuit
`openclaw/plugin-sdk/error-runtime`; core onderdrukt dan een onterechte `message_sent`-
gebeurtenis. Als een systeemeigen verzending zichtbaar werd voordat een latere bewerking mislukte,
behoud je de zichtbare subset in de fout:

```ts
import { createChannelPartialDeliveryError } from "openclaw/plugin-sdk/channel-inbound";

throw createChannelPartialDeliveryError(cause, {
  visibleReplySent: true,
  content: finalizedVisibleText,
  receipt,
});
```

Core genereert een mislukte uiteindelijke observatie met die voor de provider zichtbare inhoud en
identiteit en houdt de bezorging vervolgens op mislukt, zodat aanroepers gedeeltelijk
succes niet aanzien voor een foutloze verzending. Rapporteer `visibleReplySent: false` niet nadat een
voorbeeld, concept, bijlage of definitief bericht zichtbaar is geworden.

Wanneer `reply_payload_sending` of `message_sending` is geregistreerd, moeten die hooks
zijn afgehandeld voordat iets voor de provider zichtbaars wordt aangemaakt, omdat elke hook
de logische payload kan herschrijven of annuleren. Een voortijdig systeemeigen voorbeeld zou
inhoud van vóór de herschrijving lekken of een geannuleerd concept achterlaten. Buffer voorbeeldinhoud
totdat de geaccepteerde payload `deliver` bereikt; compatibiliteitsdispatchers die
voorbeelden eerder starten, moeten dat voortijdige voorbeeld onderdrukken zolang een van beide hooks is
geregistreerd. Gebruik voor nieuwe voorbeeldpaden de afrondbare livevoorbeeldhelpers uit
[API voor uitgaande kanalen](/nl/plugins/sdk-channel-outbound).

## Migratie

Runtime-aliassen van `runtime.channel.turn.*` zijn verwijderd. Gebruik:

- `runtime.channel.inbound.run(...)` voor onbewerkte inkomende gebeurtenissen.
- `runtime.channel.inbound.dispatchReply(...)` voor samengestelde antwoordcontexten.
- `runtime.channel.inbound.buildContext(...)` voor payloads van inkomende contexten.
- `runtime.channel.inbound.runPreparedReply(...)`, verouderd, alleen voor
  door kanalen beheerde voorbereide verzendpaden die hun eigen
  verzendclosure al samenstellen.

Nieuwe Plugincode mag geen kanaal-API's met de naam `turn` introduceren. Houd terminologie voor model- of
agentbeurten binnen agent-/providercode; kanaalplugins gebruiken termen voor inkomend verkeer,
berichten, bezorging en antwoorden.
