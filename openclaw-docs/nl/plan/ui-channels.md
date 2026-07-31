---
read_when:
    - Kanaalbericht-UI, interactieve payloads of systeemeigen kanaalrenderers refactoren
    - Mogelijkheden van berichttools, afleveringshints of contextoverschrijdende markeringen wijzigen
    - Foutopsporing voor de fan-out van Discord Carbon-imports of luie runtime-initialisatie van kanaalplugins
summary: Ontkoppel de semantische berichtpresentatie van de systeemeigen UI-renderers van kanalen.
title: Refactorplan voor kanaalpresentatie
x-i18n:
    generated_at: "2026-07-27T05:20:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## Status

Geïmplementeerd voor de gedeelde agent, CLI, Plugin-mogelijkheden en oppervlakken voor uitgaande levering:

- `ReplyPayload.presentation` bevat semantische bericht-UI.
- `ReplyPayload.delivery.pin` bevat verzoeken om verzonden berichten vast te zetten.
- Gedeelde berichtacties bieden `presentation`, `delivery` en `pin` in plaats van providerspecifieke `components`, `blocks`, `buttons` of `card`.
- De kern rendert of degradeert de presentatie automatisch via door plugins gedeclareerde mogelijkheden voor uitgaande berichten.
- Renderers voor Discord, Slack, Telegram, Mattermost, MS Teams en Feishu gebruiken het generieke contract.
- De besturingslaagcode van het Discord-kanaal importeert niet langer door Carbon ondersteunde UI-containers.

De canonieke documentatie staat nu in [Berichtpresentatie](/nl/plugins/message-presentation).
Bewaar dit plan als historische implementatiecontext; werk de canonieke handleiding bij
voor wijzigingen in contract-, renderer- of terugvalgedrag.

## Probleem

De kanaal-UI is momenteel verdeeld over meerdere incompatibele oppervlakken:

- De kern beheert via `buildCrossContextComponents` een Discord-vormige rendererhook voor meerdere contexten.
- Discord `channel.ts` kan via `DiscordUiContainer` een systeemeigen Carbon-UI importeren, waardoor UI-runtimeafhankelijkheden in de besturingslaag van de kanaalplugin terechtkomen.
- De agent en CLI bieden ontsnappingsroutes voor systeemeigen payloads, zoals Discord `components`, Slack `blocks`, Telegram of Mattermost `buttons` en Teams of Feishu `card`.
- `ReplyPayload.channelData` bevat zowel transporthints als systeemeigen UI-enveloppen.
- Het generieke `interactive`-model bestaat, maar is beperkter dan de rijkere lay-outs die Discord, Slack, Teams, Feishu, LINE, Telegram en Mattermost al gebruiken.

Hierdoor is de kern op de hoogte van systeemeigen UI-vormen, wordt het lui laden van de pluginruntime verzwakt en krijgen agents te veel providerspecifieke manieren om dezelfde berichtintentie uit te drukken.

## Doelen

- De kern bepaalt op basis van gedeclareerde mogelijkheden de beste semantische presentatie voor een bericht.
- Plugins declareren mogelijkheden en renderen semantische presentaties naar systeemeigen transportpayloads.
- De Web Control UI blijft gescheiden van systeemeigen chat-UI.
- Systeemeigen kanaalpayloads worden niet beschikbaar gesteld via het gedeelde berichtoppervlak van de agent of CLI.
- Niet-ondersteunde presentatiefuncties degraderen automatisch naar de beste tekstweergave.
- Leveringsgedrag, zoals het vastzetten van een verzonden bericht, is generieke leveringsmetadata en geen presentatie.

## Geen doelen

- Geen compatibiliteitsshim voor `buildCrossContextComponents`.
- Geen openbare systeemeigen ontsnappingsroutes voor `components`, `blocks`, `buttons` of `card`.
- Geen imports van kanaalspecifieke UI-bibliotheken in de kern.
- Geen providerspecifieke SDK-koppelingen voor gebundelde kanalen.

## Doelmodel

Voeg een door de kern beheerd veld `presentation` toe aan `ReplyPayload`.

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

`interactive` wordt tijdens de migratie een subset van `presentation`:

- `interactive`-tekstblok wordt toegewezen aan `presentation.blocks[].type = "text"`.
- `interactive`-knoppenblok wordt toegewezen aan `presentation.blocks[].type = "buttons"`.
- `interactive`-selectieblok wordt toegewezen aan `presentation.blocks[].type = "select"`.

De externe schema's voor de agent en CLI gebruiken nu `presentation`; `interactive` blijft een interne verouderde parser-/rendererhelper voor bestaande antwoordproducenten.
De openbare producentgerichte API behandelt `interactive` als verouderd. Runtime-
ondersteuning blijft bestaan, zodat bestaande goedkeuringshelpers en oudere plugins blijven
werken terwijl nieuwe code `presentation` uitvoert.

## Leveringsmetadata

Voeg een door de kern beheerd veld `delivery` toe voor verzendgedrag dat geen UI is.

```ts
type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

Semantiek:

- `delivery.pin = true` betekent dat het eerste succesvol geleverde bericht wordt vastgezet.
- `notify` is standaard `false`.
- `required` is standaard `false`; niet-ondersteunde kanalen of mislukt vastzetten degraderen automatisch door de levering voort te zetten.
- Handmatige berichtacties `pin`, `unpin` en `list-pins` blijven beschikbaar voor bestaande berichten.

De huidige binding van Telegram-ACP-onderwerpen moet van `channelData.telegram.pin = true` naar `delivery.pin = true` worden verplaatst.

## Contract voor runtimemogelijkheden

Voeg presentatie- en leveringsrendererhooks toe aan de uitgaande runtimeadapter, niet aan de kanaalplugin voor de besturingslaag.

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

Kerngedrag:

- Bepaal het doelkanaal en de runtimeadapter.
- Vraag de presentatiemogelijkheden op.
- Degradeer niet-ondersteunde blokken en pas generieke mogelijkheidslimieten toe vóór het
  renderen.
- Roep `renderPresentation` aan.
- Als er geen renderer bestaat, converteer je de presentatie naar een tekstterugval.
- Roep na een geslaagde verzending `pinDeliveredMessage` aan wanneer `delivery.pin` is aangevraagd en wordt ondersteund.

## Kanaaltoewijzing

Discord:

- Render `presentation` naar componenten v2 en Carbon-containers in modules die uitsluitend tijdens runtime worden gebruikt.
- Behoud helpers voor accentkleuren in lichte modules.
- Verwijder imports van `DiscordUiContainer` uit de besturingslaagcode van de kanaalplugin.

Slack:

- Render `presentation` naar Block Kit.
- Verwijder de invoer `blocks` uit de agent en CLI.

Telegram:

- Render tekst, context en scheidingslijnen als tekst.
- Render acties en selecties als inline toetsenborden wanneer dit is geconfigureerd en toegestaan voor het doeloppervlak.
- Gebruik een tekstterugval wanneer inline knoppen zijn uitgeschakeld.
- Verplaats het vastzetten van ACP-onderwerpen naar `delivery.pin`.

Mattermost:

- Render acties als interactieve knoppen wanneer dit is geconfigureerd.
- Render andere blokken als tekstterugval.

MS Teams:

- Render `presentation` naar Adaptive Cards.
- Behoud handmatige acties voor vastzetten, losmaken en het weergeven van vastgezette berichten.
- Implementeer eventueel `pinDeliveredMessage` als Graph-ondersteuning betrouwbaar is voor het doelgesprek.

Feishu:

- Render `presentation` naar interactieve kaarten.
- Behoud handmatige acties voor vastzetten, losmaken en het weergeven van vastgezette berichten.
- Implementeer eventueel `pinDeliveredMessage` voor het vastzetten van verzonden berichten als het API-gedrag betrouwbaar is.

LINE:

- Render `presentation` waar mogelijk naar Flex- of sjabloonberichten.
- Val voor niet-ondersteunde blokken terug op tekst.
- Verwijder LINE-UI-payloads uit `channelData`.

Eenvoudige of beperkte kanalen:

- Converteer de presentatie naar tekst met conservatieve opmaak.

## Refactorstappen

1. Pas de Discord-releasefix opnieuw toe die `ui-colors.ts` scheidt van door Carbon ondersteunde UI en `DiscordUiContainer` uit `extensions/discord/src/channel.ts` verwijdert.
2. Voeg `presentation` en `delivery` toe aan `ReplyPayload`, normalisatie van uitgaande payloads, leveringsoverzichten en hookpayloads.
3. Voeg het `MessagePresentation`-schema en parserhelpers toe in een beperkt SDK-/runtimesubpad.
4. Vervang de berichtmogelijkheden `buttons`, `cards`, `components` en `blocks` door semantische presentatiemogelijkheden.
5. Voeg hooks aan de uitgaande runtimeadapter toe voor presentatierendering en het vastzetten bij levering.
6. Vervang componentconstructie voor meerdere contexten door `buildCrossContextPresentation`.
7. Verwijder `src/infra/outbound/channel-adapters.ts` en verwijder `buildCrossContextComponents` uit kanaalplugintypen.
8. Wijzig `maybeApplyCrossContextMarker` zodat `presentation` wordt gekoppeld in plaats van systeemeigen parameters.
9. Werk verzendpaden voor plugin-dispatch bij zodat ze alleen semantische presentatie- en leveringsmetadata gebruiken.
10. Verwijder systeemeigen payloadparameters uit de agent en CLI: `components`, `blocks`, `buttons` en `card`.
11. Verwijder SDK-helpers die systeemeigen schema's voor berichttools maken en vervang ze door helpers voor presentatieschema's.
12. Verwijder UI-/systeemeigen enveloppen uit `channelData`; behoud alleen transportmetadata totdat elk resterend veld is beoordeeld.
13. Migreer renderers voor Discord, Slack, Telegram, Mattermost, MS Teams, Feishu en LINE.
14. Werk de documentatie bij voor de berichten-CLI, kanaalpagina's, plugin-SDK en de cookbook voor mogelijkheden.
15. Voer importfan-outprofilering uit voor Discord en de betrokken kanaalingangspunten.

Stappen 1-11 en 13-14 zijn in deze refactor geïmplementeerd voor de gedeelde agent, CLI, Plugin-mogelijkheden en contracten voor uitgaande adapters. Stap 12 blijft een diepere interne opschoningsronde voor providerprivate `channelData`-transportenveloppen. Stap 15 blijft vervolgvalidatie als we gekwantificeerde importfan-outcijfers willen naast de type-/testgate.

## Tests

Toevoegen of bijwerken:

- Tests voor presentatienormalisatie.
- Tests voor automatische degradatie van presentaties bij niet-ondersteunde blokken.
- Markeringstests voor meerdere contexten voor plugin-dispatch en kernleveringspaden.
- Tests voor de kanaalrendermatrix voor Discord, Slack, Telegram, Mattermost, MS Teams, Feishu, LINE en tekstterugval.
- Schematests voor berichttools die aantonen dat systeemeigen velden zijn verwijderd.
- CLI-tests die aantonen dat systeemeigen vlaggen zijn verwijderd.
- Regressietest voor lui importeren bij het Discord-ingangspunt, met dekking voor Carbon.
- Tests voor vastzetten bij levering, met dekking voor Telegram en generieke terugval.

## Open vragen

- Moet `delivery.pin` in de eerste fase worden geïmplementeerd voor Discord, Slack, MS Teams en Feishu, of eerst alleen voor Telegram?
- Moet `delivery` uiteindelijk bestaande velden zoals `replyToId`, `replyToCurrent`, `silent` en `audioAsVoice` opnemen, of gericht blijven op gedrag na het verzenden?
- Moet de presentatie afbeeldingen of bestandsverwijzingen rechtstreeks ondersteunen, of moeten media voorlopig gescheiden blijven van de UI-lay-out?

## Gerelateerd

- [Overzicht van kanalen](/nl/channels)
- [Berichtpresentatie](/nl/plugins/message-presentation)
