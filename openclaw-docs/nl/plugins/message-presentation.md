---
read_when:
    - Rendering van berichtkaarten, grafieken, tabellen, knoppen of selecties toevoegen of wijzigen
    - Een kanaalplugin bouwen die uitgebreide uitgaande berichten ondersteunt
    - Presentatie- of bezorgmogelijkheden van de berichtentool wijzigen
    - Fouten opsporen in providerspecifieke regressies bij de weergave van kaarten/blokken/componenten
summary: Semantische berichtkaarten, grafieken, tabellen, bedieningselementen, terugvaltekst en afleveringsaanwijzingen voor kanaalplugins
title: Berichtweergave
x-i18n:
    generated_at: "2026-07-27T05:58:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

Berichtpresentatie is het gedeelde contract van OpenClaw voor rijke uitgaande chatinterfaces.
Hiermee kunnen agents, CLI-opdrachten, goedkeuringsflows en plugins de intentie van
het bericht één keer beschrijven, terwijl elke kanaalplugin de best mogelijke
native vorm rendert.

Gebruik presentatie voor overdraagbare berichtinterfaces: tekstsecties, kleine
context-/voetteksten, scheidingslijnen, grafieken, tabellen, knoppen, selectiemenu's
en kaarttitels/-tonen.

Voeg geen nieuwe providerspecifieke native velden zoals Discord `components`, Slack
`blocks`, Telegram `buttons`, Teams `card` of Feishu `card` toe aan de gedeelde
berichttool. Dit zijn rendereruitvoeren die eigendom zijn van de kanaalplugin.

## Contract

Pluginauteurs importeren het openbare contract uit:

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

Structuur:

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** Verouderde callbackwaarde. Geef voor nieuwe besturingselementen de voorkeur aan action. */
  value?: string;
  /** @deprecated Gebruik een action met type "url". */
  url?: string;
  /** @deprecated Gebruik een action met type "web-app". */
  webApp?: { url: string };
  /** @deprecated Gebruik een action met type "web-app". */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** Verouderde callbackwaarde. Geef voor nieuwe besturingselementen de voorkeur aan action. */
  value?: string;
};

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

Semantiek van knoppen:

- `action.type: "command"` voert een native slash-opdracht uit via het opdrachtpad
  van de kern. Gebruik dit voor ingebouwde opdrachtknoppen en menu's.
- `action.type: "callback"` voert ondoorzichtige plugingegevens door het interactiepad
  van het kanaal. Kanaalplugins mogen callbackgegevens niet opnieuw interpreteren
  als slash-opdrachten.
- `action.type: "approval"` identificeert één duurzame goedkeuring door een operator, het
  expliciete type `exec` of `plugin` en de gevraagde beslissing. Kanaalplugins
  coderen die actie in een transportspecifieke privé-callback en verwerken deze
  via de goedkeuringsservice; ze mogen geen `/approve`-opdrachttekst parseren of
  het type uit de ID afleiden.
- `action.type: "question"` identificeert één keuze voor een actieve, tijdens runtime opgestelde
  `ask_user`-vraag. Net als `approval` is dit een OpenClaw-runtimeactie;
  agents en plugins mogen geen vraag-ID's genereren. Telegram, Discord en
  Slack zetten deze om in transportspecifieke privé-native callbacks en verwerken
  de keuze via de Gateway. Wanneer de vraag is beantwoord, verlopen of
  geannuleerd, bewerken die kanalen het afgeleverde bericht, verwijderen ze de acties
  en voegen ze de eindstatus toe. WhatsApp, Signal en iMessage renderen maximaal
  vier enkelvoudige selectiekeuzes als reacties van `1️⃣` tot en met `4️⃣`. Andere
  vraagvormen worden teruggebracht tot labeltekst en de gebruiker kan antwoorden
  met een bericht in platte tekst.
- `action.type: "url"` opent een normale link.
- `action.type: "web-app"` start een kanaalspecifieke native webapp. Stel `url` in voor een
  URL-gebaseerde app of `widgetId` voor een door OpenClaw gehoste widget waarvan het
  startmechanisme eigendom is van het kanaal; ten minste één ervan is vereist. Wanneer
  beide aanwezig zijn, kan een kanaal de voorkeur geven aan het native startmechanisme
  voor gehoste widgets en de URL gebruiken waar dat mechanisme niet beschikbaar is.
- `value` is de verouderde ondoorzichtige callbackwaarde. Nieuwe besturingselementen moeten `action`
  gebruiken, zodat kanaalplugins opdrachten en callbacks kunnen omzetten zonder
  op basis van tekst te hoeven gokken.
- `url`, `webApp` en `web_app` blijven geaccepteerd als verouderde invoer aan de grens.
  Normalisatiefuncties behouden deze velden, zodat renderers onderscheid kunnen maken
  tussen uitgebrachte verouderde semantiek en expliciete getypeerde acties. Nieuwe producenten
  moeten `action` gebruiken.
- `label` is vereist en wordt ook gebruikt in de tekstuele fallback.
- `style` is adviserend. Renderers moeten niet-ondersteunde stijlen omzetten naar een
  veilige standaardwaarde en het verzenden niet laten mislukken.
- `priority` is optioneel. Wanneer een kanaal actielimieten bekendmaakt en besturingselementen
  moeten worden verwijderd, behoudt de kern eerst de knoppen met een hogere prioriteit en
  de oorspronkelijke volgorde van knoppen met dezelfde prioriteit. Wanneer alle
  besturingselementen passen, blijft de opgestelde volgorde behouden.
- `disabled` is optioneel. Kanalen moeten zich aanmelden met `supportsDisabled`; anders
  brengt de kern het uitgeschakelde besturingselement terug tot niet-interactieve fallbacktekst.
  Een uitgeschakelde knop wordt in fallbacktekst altijd alleen als label weergegeven, zelfs
  wanneer deze een `command`-actie bevat.
- `reusable` is optioneel. Kanalen die herbruikbare native callbacks ondersteunen, mogen
  de actie na een geslaagde interactie beschikbaar houden. Gebruik dit voor
  herhaalbare of idempotente acties zoals vernieuwen, inspecteren of meer details;
  laat dit uitgeschakeld voor normale eenmalige goedkeuringen en destructieve acties.

Semantiek van selecties:

- `options[].action` accepteert alleen `command` of `callback`; goedkeurings- en linkacties zijn alleen voor knoppen.
- `options[].value` is de verouderde geselecteerde toepassingswaarde.
- `placeholder` is adviserend en kan worden genegeerd door kanalen zonder native
  selectieondersteuning.
- Als een kanaal geen selecties ondersteunt, vermeldt de fallbacktekst de labels.

Semantiek van grafieken:

- `pie` vereist positieve segmentwaarden.
- `bar`, `area` en `line` gebruiken één geordende `categories`-array. Elke reeks
  levert precies één eindige waarde per categorie, in dezelfde volgorde.
- Categorielabels en reeksnamen moeten uniek zijn. Ongeldige of onvolledige
  grafiekblokken worden tijdens normalisatie verwijderd in plaats van de gegevens
  stilzwijgend te wijzigen.
- Native grafiekweergave vereist aanmelding via `presentationCapabilities.charts`.
  Andere kanalen ontvangen de grafiektitel, assen, categorieën, reeksen en waarden
  als deterministische tekst. Dit is ook de toegankelijkheidsfallback.

Semantiek van tabellen:

- `caption` is een vereiste korte kop. `headers` moet ten minste één
  uniek, niet-leeg kolomlabel bevatten.
- `rows` moet ten minste één rij bevatten. Elke rij moet precies één cel per
  kop hebben en elke cel moet een niet-lege tekenreeks of een eindig getal zijn.
- `rowHeaderColumnIndex` is een optionele op nul gebaseerde index die de kolom identificeert
  waarvan de cellen door native renderers als rijkoppen moeten worden weergegeven.
- Tabelnormalisatie is atomair. Bij een ongeldig bijschrift, een ongeldige kop,
  rijbreedte, cel of rijkopindex wordt het tabelblok verwijderd in plaats van dat
  de gegevens worden afgekapt of hersteld.
- Native tabelweergave vereist aanmelding via `presentationCapabilities.tables`.
  Andere kanalen ontvangen het bijschrift en elke rij als deterministische lineaire
  tekst, waarbij interne witruimte wordt samengevouwen:

  ```text
  Open pijplijn (tabel)
  - Account: Acme; Fase: Gewonnen; ARR: 125000
  - Account: Globex; Fase: Beoordeling; ARR: 82000
  ```

Er is geen afzonderlijke `report`-discriminator. Stel een rapport samen uit `title`,
`tone`, `text`, `context`, `chart`, `table` en actieblokken. Hierdoor blijft elk
blok afzonderlijk renderbaar en krijgt het volledige rapport dezelfde
deterministische tekstuele fallback.

## Voorbeelden voor producenten

Eenvoudige kaart:

```json
{
  "title": "Implementatiegoedkeuring",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "Canary kan worden gepromoveerd." },
    { "type": "context", "text": "Build 1234, staging geslaagd." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Goedkeuren",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "Afwijzen",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

Linkknop met alleen een URL:

```json
{
  "blocks": [
    { "type": "text", "text": "De releaseopmerkingen zijn klaar." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Opmerkingen openen",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

Knop voor Telegram Mini App:

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Starten",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

Selectiemenu:

```json
{
  "title": "Omgeving kiezen",
  "blocks": [
    {
      "type": "select",
      "placeholder": "Omgeving",
      "options": [
        { "label": "Canary", "value": "env:canary" },
        { "label": "Productie", "value": "env:prod" }
      ]
    }
  ]
}
```

Grafiek:

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "Kwartaalomzet",
      "categories": ["Q1", "Q2", "Q3"],
      "series": [
        { "name": "Product", "values": [120, 145, 138] },
        { "name": "Diensten", "values": [80, 95, 104] }
      ],
      "xLabel": "Kwartaal",
      "yLabel": "Omzet"
    }
  ]
}
```

Tabelrapport:

```json
{
  "title": "Pijplijnrapport",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "Huidige verkoopkansen per fase." },
    {
      "type": "table",
      "caption": "Open pijplijn",
      "headers": ["Account", "Fase", "ARR"],
      "rows": [
        ["Acme", "Gewonnen", 125000],
        ["Globex", "Beoordeling", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "Bijgewerkt vanuit de CRM-momentopname." }
  ]
}
```

Verzenden via CLI:

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "Implementatiegoedkeuring" \
  --presentation '{"title":"Implementatiegoedkeuring","tone":"warning","blocks":[{"type":"text","text":"Canary is klaar."},{"type":"buttons","buttons":[{"label":"Goedkeuren","value":"deploy:approve","style":"success"},{"label":"Afwijzen","value":"deploy:decline","style":"danger"}]}]}'
```

Vastgezette aflevering:

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "Onderwerp geopend" \
  --pin
```

Vastgezette bezorging met expliciete JSON:

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## Renderercontract

Kanaalplugins declareren rendererondersteuning op hun uitgaande adapter:

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

Capability-booleans beschrijven wat de renderer interactief kan maken. Optionele
`limits` beschrijven de generieke envelop die de kern kan aanpassen voordat de
renderer wordt aangeroepen:

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
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
```

De kern past generieke limieten toe op semantische bedieningselementen vóór het renderen. Renderers
blijven verantwoordelijk voor de uiteindelijke providerspecifieke validatie en afkapping voor het aantal native blokken,
de kaartgrootte, URL-limieten en provider-eigenaardigheden die niet in
het generieke contract kunnen worden uitgedrukt. Als limieten alle bedieningselementen uit een blok verwijderen, behoudt de kern
de labels als niet-interactieve contexttekst, zodat het bezorgde bericht nog steeds een
zichtbaar alternatief heeft.

## Kernrenderflow

Op het canonieke uitgaande pad dat door de CLI en standaardberichtacties wordt gebruikt, doet de kern het volgende:

1. Normaliseert de presentatiepayload.
2. Bepaalt de uitgaande adapter van het doelkanaal.
3. Leest `presentationCapabilities`.
4. Past generieke capability-limieten toe, zoals het aantal acties, de labellengte en
   het aantal selectieopties, wanneer de adapter deze adverteert. Grafiek- en tabelblokken
   worden deterministische tekst, tenzij de adapter respectievelijk expliciet
   `charts: true` of `tables: true` adverteert.
5. Roept `renderPresentation` aan wanneer de adapter de payload kan renderen.
6. Valt terug op conservatieve tekst wanneer de adapter ontbreekt of niet kan renderen.
7. Verzendt de resulterende payload via het normale bezorgingspad van het kanaal.
8. Past bezorgingsmetadata zoals `delivery.pin` toe na het eerste succesvol
   verzonden bericht.

Kanaallokale antwoord- of voorbeeldflows die `ReplyPayload` rechtstreeks verwerken,
moeten ofwel dat canonieke pad volgen, of hetzelfde presentatiealternatief materialiseren
voordat de payload wordt teruggebracht tot platte tekst/media.

De kern is verantwoordelijk voor het terugvalgedrag, zodat producenten kanaalonafhankelijk kunnen blijven. Kanaalplugins
zijn verantwoordelijk voor native rendering en interactieafhandeling.

## Degradatieregels

De presentatie moet veilig kunnen worden verzonden via beperkte kanalen.

Alternatieve tekst bevat:

- `title` als eerste regel
- `text`-blokken als normale alinea's
- `context`-blokken als compacte contextregels
- `divider`-blokken als visueel scheidingsteken
- knoplabels, inclusief URL's voor linkknoppen
- labels van selectieopties
- grafiektitel, -type, assen, categorieën, reeksen en waarden
- tabelbijschrift, kopteksten en elke rijwaarde

### Zichtbaarheid van knopwaarden in het alternatief

Wanneer een kanaal geen interactieve bedieningselementen kan renderen, vallen knop- en selectiewaarden
terug op platte tekst. Het terugvalgedrag behoudt de bruikbaarheid en
houdt tegelijkertijd ondoorzichtige callbackgegevens privé:

- **Acties van het type `command`** worden gerenderd als `` label: `command` ``, zodat gebruikers
  de opdracht kunnen kopiëren en handmatig in de kanaalinvoer kunnen uitvoeren.
- **Acties van het type `callback`** en verouderde **`value`**-velden worden
  uitsluitend als label gerenderd. De ondoorzichtige callbackwaarde wordt niet weergegeven in de alternatieve tekst.
- **Acties van het type `approval`** worden uitsluitend als label gerenderd. Goedkeurings-ID's en beslissingen zijn
  transportgegevens en worden niet via generieke scalaire helpers of alternatieve
  tekst weergegeven.
- **`url`-acties**, door URL's ondersteunde **`web-app`-acties** en verouderde **`url` /
  `webApp` / `web_app`**-invoer renderen de URL-tekst naast het knoplabel,
  omdat de URL voor de gebruiker zichtbaar is. Acties die uitsluitend voor gehoste widgets zijn bedoeld, worden alleen als label weergegeven op
  kanalen zonder native widgetstart.
- **Selectieopties** worden uitsluitend als label gerenderd. De onderliggende optiewaarde wordt niet
  weergegeven in de alternatieve tekst.

Kanaaladapters die instructies voor handmatige opdrachten toevoegen aan hun alternatieve UI (bijvoorbeeld
instructies voor Feishu-documentopmerkingen), moeten de controle op de aanwezigheid van opdrachten afleiden
uit dezelfde presentatieblokken die de alternatieve renderer gebruikt, zodat de
instructietekst alleen verschijnt wanneer er daadwerkelijk een handmatige opdracht wordt weergegeven.

Niet-ondersteunde native bedieningselementen moeten degraderen in plaats van de volledige verzending te laten mislukken.
Voorbeelden:

- Telegram met uitgeschakelde inlineknoppen verzendt het tekstalternatief.
- Een kanaal zonder selectieondersteuning vermeldt selectieopties als tekst.
- Een kanaal zonder native grafiekondersteuning vermeldt de grafiekgegevens als tekst.
- Een kanaal zonder native tabelondersteuning vermeldt elke tabelrij als tekst.
- Een knop met alleen een URL wordt een native linkknop of een alternatieve URL-regel.
- Optionele fouten bij vastzetten laten het bezorgde bericht niet mislukken.

De belangrijkste uitzondering is `delivery.pin.required: true`; als vastzetten als
verplicht is aangevraagd en het kanaal het verzonden bericht niet kan vastzetten, meldt de bezorging een fout.

## Providertoewijzing

Huidige gebundelde renderers:

| Kanaal          | Native renderdoel                         | Opmerkingen                                                                                                                                                                                                       |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | Componenten en componentcontainers        | Behoudt verouderde `channelData.discord.components` voor bestaande producenten van providernative payloads, maar nieuwe gedeelde verzendingen moeten `presentation` gebruiken.                                                |
| Feishu          | Interactieve kaarten                      | De kaartkop kan `title` gebruiken; de hoofdtekst vermijdt duplicatie van die titel.                                                                                                                     |
| Matrix          | Tekstalternatief plus gestructureerd gebeurtenisveld | Knoppen/selecties worden als ondersteund geadverteerd, maar elk blok wordt momenteel gerenderd als `renderMessagePresentationFallbackText`-uitvoer in een `com.openclaw.presentation`-gebeurtenisveld, niet als native interactieve widgets. |
| Mattermost      | Tekst plus interactieve eigenschappen     | Selecties en scheidingstekens worden niet ondersteund; die blokken degraderen tot tekst.                                                                                                                           |
| Microsoft Teams | Adaptive Cards                            | Platte `message`-tekst wordt samen met de kaart opgenomen wanneer beide worden aangeleverd. Selecties, stijlen en uitgeschakelde status worden niet ondersteund.                                           |
| Slack           | Block Kit                                 | Rendert `chart` als native `data_visualization` en `table` als native `data_table`; behoudt verouderde `channelData.slack.blocks`, maar nieuwe gedeelde verzendingen moeten `presentation` gebruiken. |
| Telegram        | Tekst plus inline-toetsenborden           | Knoppen/selecties vereisen inlineknop-capability voor het doeloppervlak; anders wordt het tekstalternatief gebruikt.                                                                                               |
| Platte kanalen  | Tekstalternatief                          | Kanalen zonder renderer krijgen nog steeds leesbare uitvoer.                                                                                                                                                      |

Compatibiliteit met providernative payloads is een overgangsvoorziening voor bestaande
antwoordproducenten. Dit is geen reden om nieuwe gedeelde native velden toe te voegen.

## Presentation versus InteractiveReply

`InteractiveReply` is de oudere interne subset die door goedkeurings- en interactiehelpers wordt gebruikt.
Deze ondersteunt:

- tekst
- knoppen
- selecties

`MessagePresentation` is het canonieke gedeelde verzendcontract. Dit voegt het volgende toe:

- titel
- toon
- context
- scheidingsteken
- grafiek
- tabel
- knoppen met alleen een URL
- generieke bezorgingsmetadata via `ReplyPayload.delivery`

Gebruik helpers uit `openclaw/plugin-sdk/interactive-runtime` bij het overbruggen van oudere
code:

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

Nieuwe code moet `MessagePresentation` rechtstreeks accepteren of produceren. Bestaande
`interactive`-payloads zijn een verouderde subset van `presentation`; runtime-
ondersteuning blijft beschikbaar voor oudere producenten.

Niet-verouderde helpers die nuttig zijn om te kennen:

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  valideren en zetten een ongetypeerde payload (bijvoorbeeld JSON van de CLI-vlag
  `--presentation`) om naar `MessagePresentation`.
- `isMessagePresentationInteractiveBlock(block)` beperkt een blok tot de
  unie `buttons` | `select`.
- `resolveMessagePresentationButtonAction(button)` en
  `resolveMessagePresentationOptionAction(option)` retourneren de canonieke getypeerde
  actie en accepteren daarbij verouderde grensvelden. Een expliciete `action`
  heeft altijd voorrang.
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` lezen uitsluitend scalaire
  opdracht-/callbackwaarden. Een niet-scalaire canonieke actie valt nooit terug op een
  verouderde schaduw-`value`, zodat goedkeurings-ID's en linkdoelen getypeerd blijven.
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)` geven één gestructureerd
  gegevensblok weer als deterministische tekst voor kanaalspecifieke fallbackpaden.

De verouderde `InteractiveReply*`-typen en conversiehelpers zijn in de SDK gemarkeerd als
`@deprecated`:

- `InteractiveReply`, `InteractiveReplyBlock`, `InteractiveReplyButton` en
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` en
`presentationToInteractiveControlsReply(...)` blijven beschikbaar als rendererbruggen
voor verouderde kanaalimplementaties. Nieuwe producercode hoort ze niet aan te roepen;
stuur `presentation` en laat de aanpassing door de kern/het kanaal de weergave afhandelen.

Goedkeuringshelpers hebben ook presentatiegerichte vervangingen:

- gebruik `buildApprovalPresentation(...)` in plaats van
  `buildApprovalInteractiveReply(...)`
- gebruik `buildExecApprovalPresentation(...)` in plaats van
  `buildExecApprovalInteractiveReply(...)`

Die uitgebrachte builders blijven voor Plugin-compatibiliteit op opdrachten gebaseerd. Gateway-
en gebundelde kanaalcode die eigenaar is van een duurzaam goedkeuringstype, hoort
`buildTypedApprovalPresentation(...)`,
`buildTypedExecApprovalPendingReplyPayload(...)` of
`buildTypedPluginApprovalPendingReplyPayload(...)` te gebruiken, zodat transports een
expliciete `approval`-actie ontvangen in plaats van semantiek af te leiden uit `/approve`-tekst.

`renderMessagePresentationFallbackText(...)` retourneert een lege tekenreeks voor
presentatieblokken die geen tekstuele fallback hebben, zoals een presentatie met
alleen een scheidingslijn. Transports die een niet-lege verzendinhoud vereisen, kunnen
`emptyFallback` doorgeven om te kiezen voor een minimale inhoud zonder het standaard
fallbackcontract te wijzigen.

## Vastzetten bij aflevering

Vastzetten is afleveringsgedrag, geen presentatie. Gebruik `delivery.pin` in plaats van
providerspecifieke velden zoals `channelData.telegram.pin`.

Semantiek:

- `pin: true` zet het eerste succesvol afgeleverde bericht vast.
- `pin.notify` is standaard `false`.
- `pin.required` is standaard `false`.
- Optionele fouten bij het vastzetten leiden tot degradatie en laten het verzonden bericht intact.
- Verplichte fouten bij het vastzetten laten de aflevering mislukken.
- Bij berichten in delen wordt het eerste afgeleverde deel vastgezet, niet het laatste deel.

Handmatige berichtacties `pin`, `unpin` en `pins` blijven bestaan voor bestaande
berichten waarvoor de provider deze bewerkingen ondersteunt.

## Checklist voor Plugin-auteurs

- Declareer `presentation` vanuit `describeMessageTool(...)` wanneer het kanaal
  semantische presentatie kan weergeven of veilig kan degraderen.
- Voeg `presentationCapabilities` toe aan de uitgaande runtime-adapter.
- Implementeer `renderPresentation` in runtimecode, niet in de
  Plugin-installatiecode van het besturingsvlak.
- Houd native UI-bibliotheken buiten intensief gebruikte installatie-/cataloguspaden.
- Declareer generieke capaciteitslimieten op `presentationCapabilities.limits` wanneer
  ze bekend zijn.
- Handhaaf de uiteindelijke platformlimieten in de renderer en tests.
- Voeg fallbacktests toe voor niet-ondersteunde grafieken, tabellen, knoppen, selecties, URL-
  knoppen, duplicatie van titel/tekst en gemengde verzendingen met `message` plus `presentation`.
- Voeg ondersteuning voor vastzetten bij aflevering uitsluitend toe via `deliveryCapabilities.pin` en
  `pinDeliveredMessage` wanneer de provider de ID van het verzonden bericht kan vastzetten.
- Stel geen nieuwe providerspecifieke kaart-/blok-/component-/knopvelden beschikbaar via
  het gedeelde schema voor berichtacties.

## Gerelateerde documentatie

- [Berichten-CLI](/nl/cli/message)
- [Overzicht van de Plugin-SDK](/nl/plugins/sdk-overview)
- [Plugin-architectuur](/nl/plugins/architecture-internals#message-tool-schemas)
- [Refactorplan voor kanaalpresentatie](/nl/plan/ui-channels)
