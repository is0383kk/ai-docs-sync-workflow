---
read_when:
    - Uitleg over hoe inkomende berichten worden omgezet in antwoorden
    - Sessies, wachtrijmodi of streaminggedrag verduidelijken
    - Documentatie over de zichtbaarheid van redeneringen en de gevolgen voor het gebruik
summary: Berichtenstroom, sessies, wachtrijen en zichtbaarheid van redeneringen
title: Berichten
x-i18n:
    generated_at: "2026-07-27T04:57:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e42bed834e9a57fb8a248c8654b75ea9977928582f68a83859cf6c16ed0b6bf5
    source_path: concepts/messages.md
    workflow: 16
---

Binnenkomende berichten doorlopen routering, deduplicatie/debounce, een agentrun en uitgaande aflevering:

```text
Binnenkomend bericht
  -> routering/bindingen -> sessiesleutel
  -> deduplicatie + debounce
  -> wachtrij (als er al een run actief is)
  -> agentrun (streaming + tools)
  -> uitgaande antwoorden (kanaallimieten + opdelen)
```

Belangrijkste configuratieonderdelen:

- `messages.*` voor voorvoegsels, wachtrijen, debounce van binnenkomende berichten en groepsgedrag.
- `agents.defaults.*` voor blokstreaming, opdelen en standaardinstellingen voor stille antwoorden.
- Kanaaloverschrijvingen (`channels.telegram.*`, `channels.whatsapp.*`, enz.) voor limieten en streamingopties per kanaal.

Zie [Configuratie](/nl/gateway/configuration) voor het volledige schema.

## Deduplicatie van binnenkomende berichten

Kanalen kunnen na een nieuwe verbinding hetzelfde bericht opnieuw afleveren. OpenClaw bewaart een cache in het geheugen, met als sleutel het agentbereik, de kanaalroute (kanaal + peer + account + thread) en het bericht-ID, zodat een opnieuw afgeleverd bericht geen tweede agentrun activeert. De cachevermelding verloopt na 20 minuten of zodra 5000 vermeldingen worden bijgehouden, afhankelijk van wat het eerst gebeurt.

## Debounce van binnenkomende berichten

Snel opeenvolgende tekstberichten van dezelfde afzender kunnen via `messages.inbound` worden gebundeld tot één agentbeurt. Debounce is beperkt tot het betreffende kanaal + gesprek en gebruikt het meest recente bericht voor antwoordthreads/ID's.

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,
      byChannel: {
        discord: 1500,
        slack: 1500,
        whatsapp: 5000,
      },
    },
  },
}
```

- Debounce is alleen van toepassing op tekstberichten; media/bijlagen worden onmiddellijk verwerkt.
- Besturingsopdrachten (stop/abort/status enz.) omzeilen debounce, zodat ze onmiddellijk worden uitgevoerd.
- Standaard uitgeschakeld: `messages.inbound.debounceMs` heeft geen ingebouwde standaardwaarde, dus debounce wordt pas geactiveerd nadat je dit instelt (globaal of per kanaal).
- iMessage volgt hetzelfde algemene debouncebeleid. `imsg` 0.13.1 en nieuwer voegt opgesplitste verzendingen van Apple URL-voorvertoningen samen voordat OpenClaw ze ontvangt, dus er is geen iMessage-specifieke debounce-instelling nodig.

## Sessies en apparaten

Sessies zijn eigendom van de Gateway, niet van clients.

- Directe chats worden samengevoegd onder de hoofdsessiesleutel van de agent.
- Groepen/kanalen krijgen hun eigen sessiesleutels.
- De sessieopslag en transcripties bevinden zich op de Gateway-host.

Meerdere apparaten/kanalen kunnen aan dezelfde sessie worden gekoppeld, maar de geschiedenis wordt niet volledig naar elke client teruggesynchroniseerd. Gebruik één primair apparaat voor lange gesprekken om uiteenlopende context te voorkomen. De Control UI en TUI tonen altijd het door de Gateway beheerde sessietranscript en vormen dus de bron van waarheid.

Details: [Sessiebeheer](/nl/concepts/session).

## Promptteksten en geschiedeniscontext

Kanaalplugins vullen verschillende tekstvelden in de binnenkomende context, van hoogste naar laagste voorkeur:

| Veld              | Doel                                                                                                        |
| ----------------- | ----------------------------------------------------------------------------------------------------------- |
| `BodyForAgent`    | Modelgerichte tekst voor de huidige beurt. Valt terug op `CommandBody` / `RawBody` / `Body` wanneer niet ingesteld.        |
| `BodyForCommands` | Schone tekst voor het parseren van instructies/opdrachten. Valt terug op `CommandBody` / `RawBody` / `Body` wanneer niet ingesteld. |
| `CommandBody`     | Verouderde tussentijdse berichttekst; geef de voorkeur aan `BodyForCommands`.                               |
| `RawBody`         | Afgeschafte alias voor `CommandBody`.                                                                  |
| `Body`            | Verouderde prompttekst; kan kanaalenveloppen en geschiedeniswrappers bevatten.                              |

Wanneer een kanaal geschiedenis aanlevert, wordt deze als volgt ingesloten:

- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

Voor niet-directe chats (groepen/kanalen/ruimten) krijgt de huidige berichttekst een voorvoegsel met het afzenderlabel, overeenkomstig de stijl van geschiedenisvermeldingen. Het verwijderen van instructies geldt alleen voor het gedeelte met het huidige bericht, zodat de geschiedenis intact blijft. Kanalen die geschiedenis insluiten, moeten `BodyForCommands` (of de verouderde `CommandBody` / `RawBody`) instellen op de oorspronkelijke berichttekst en `Body` als de gecombineerde prompt behouden.

Geschiedenisbuffers bevatten alleen nog niet verwerkte berichten: ze bevatten groepsberichten die geen run activeerden (bijvoorbeeld berichten waarvoor een vermelding vereist is) en sluiten berichten uit die al in het sessietranscript staan. Gestructureerde geschiedenis-, antwoord-, doorgestuurde en kanaalmetadata worden tijdens het samenstellen van de prompt weergegeven als niet-vertrouwde contextblokken met de gebruikersrol.

Configureer de omvang van de geschiedenis met `messages.groupChat.historyLimit` (globale standaardwaarde) of kanaalspecifieke overschrijvingen zoals `channels.slack.historyLimit` en `channels.telegram.accounts.<id>.historyLimit` (stel `0` in om dit uit te schakelen).

## Metadata van toolresultaten

`content` van een toolresultaat is het voor het model zichtbare resultaat; `details` is runtimemetadata voor UI-weergave, diagnostiek, medialevering en plugins.

- `toolResult.details` wordt verwijderd vóór herhaling door de provider en vóór invoer voor Compaction.
- Opgeslagen sessietranscripties behouden alleen begrensde `details`; te grote metadata wordt vervangen door een compacte samenvatting met de markering `persistedDetailsTruncated: true`.
- Plugins en tools moeten tekst die het model moet lezen in `content` plaatsen, niet alleen in `details`.

## Wachtrijen en vervolgberichten

Wanneer er al een run actief is, sturen binnenkomende berichten die standaard bij. `messages.queue` bepaalt de modus:

| Modus             | Gedrag                                              |
| ----------------- | --------------------------------------------------- |
| `steer` (standaard) | Voeg de nieuwe prompt toe aan de actieve run.       |
| `followup`        | Voer het bericht uit nadat de actieve run is voltooid. |
| `collect`         | Bundel compatibele berichten in één latere beurt.   |
| `interrupt`       | Breek de actieve run af en start daarna de nieuwste prompt. |

De wachtrij gebruikt een ingebouwde debounce van 500ms voor bijsturen, vervolgberichten en verzamelbundeling. `messages.queue.cap` heeft standaard een maximum van 20 berichten in de wachtrij en `messages.queue.drop` is standaard `summarize` (`old` en `new` zijn ook beschikbaar). Configureer kanaalspecifieke overschrijvingen via `messages.queue.byChannel` en `messages.queue.debounceMsByChannel`.

Details: [Opdrachtwachtrij](/nl/concepts/queue) en [Bijsturingswachtrij](/nl/concepts/queue-steering).

## Eigenaarschap van kanaalruns

Kanaalplugins kunnen de volgorde behouden, invoer debouncen en transportbackpressure toepassen voordat een bericht de sessiewachtrij binnenkomt. Ze mogen geen afzonderlijke time-out opleggen rond de agentbeurt zelf. Zodra een bericht naar een sessie is gerouteerd, bepalen de levenscycli van de sessie, tools en runtime hoe langlopende werkzaamheden worden beheerd, zodat alle kanalen trage beurten consistent rapporteren en ervan herstellen.

## Streaming, opdelen en bundelen

Blokstreaming verzendt gedeeltelijke antwoorden terwijl het model tekstblokken produceert; opdelen houdt rekening met de tekstlimieten van kanalen en voorkomt het splitsen van omheinde code.

- `agents.defaults.blockStreamingDefault` (`on|off`, standaard `off`)
- `agents.defaults.blockStreamingBreak` (`text_end|message_end`)
- `agents.defaults.blockStreamingChunk` (`minChars|maxChars|breakPreference`)
- `agents.defaults.blockStreamingCoalesce` (bundeling op basis van inactiviteit)
- `agents.defaults.humanDelay` (mensachtige pauze tussen blokantwoorden)
- Kanaaloverschrijvingen: `*.streaming.block.enabled` en `*.streaming.block.coalesce` op meegeleverde kanalen; verouderde platte sleutels worden gemigreerd door `openclaw doctor --fix`. Blokstreaming is uitgeschakeld tenzij dit expliciet wordt ingeschakeld, op elk kanaal inclusief Telegram. QQ Bot is de uitzondering: deze heeft geen `streaming.block`-sleutels en streamt blokantwoorden tenzij `channels.qqbot.streaming.mode` gelijk is aan `"off"`.

Details: [Streaming + opdelen](/nl/concepts/streaming).

## Zichtbaarheid van redeneringen en tokens

- `/reasoning on|off|stream` bepaalt de zichtbaarheid.
- De inhoud van redeneringen telt nog steeds mee voor het tokengebruik wanneer het model deze produceert.
- Telegram ondersteunt het streamen van redeneringen naar een tijdelijke conceptballon die na de definitieve aflevering wordt verwijderd; gebruik `/reasoning on` voor blijvende uitvoer van redeneringen.

Details: [Instructies voor denken + redeneren](/nl/tools/thinking) en [Tokengebruik](/nl/reference/token-use).

## Voorvoegsels, threads en antwoorden

- Uitgaande voorvoegsels staan in `channels.<channel>.responsePrefix` en `channels.<channel>.accounts.<id>.responsePrefix`. Accountwaarden hebben voorrang. Doctor kopieert de globale terugvalwaarde naar geconfigureerde kanaalblokken wanneer die canonieke velden niet zijn ingesteld; `messages.responsePrefix` blijft beschikbaar als terugvalwaarde voor impliciete en aangepaste kanalen.
- Antwoordthreads via `replyToMode` en kanaalspecifieke standaardwaarden.

Details: [Configuratie](/nl/gateway/config-agents#messages) en de kanaaldocumentatie.

## Stille antwoorden

Het stille token `NO_REPLY` (niet hoofdlettergevoelig, dus `no_reply` komt ook overeen) betekent "lever geen voor de gebruiker zichtbaar antwoord af." Wanneer een beurt ook nog niet afgeleverde toolmedia bevat, zoals gegenereerde TTS-audio, verwijdert OpenClaw de stille tekst maar levert het de mediabijlage nog steeds af.

Het stiltebeleid wordt bepaald door het gesprekstype:

- Directe gesprekken ontvangen nooit `NO_REPLY`-promptinstructies. Als een directe run per ongeluk uitsluitend een stil token retourneert, onderdrukt OpenClaw dit in plaats van het te herschrijven of af te leveren.
- Groepen/kanalen staan stilte standaard toe. In de modus `message_tool` voor zichtbare antwoorden betekent stilte dat het model `message(action=send)` niet aanroept.
- Interne orkestratie staat stilte standaard toe.

De standaardwaarden staan onder `agents.defaults.silentReply`; `surfaces.<id>.silentReply` kan het groeps-/interne beleid per oppervlak overschrijven.

OpenClaw gebruikt stille antwoorden ook voor algemene interne fouten van de runner in niet-directe chats, zodat groepen/kanalen geen standaardtekst over Gateway-fouten zien. Geclassificeerde fouten met voor de gebruiker zichtbare herstelinformatie, zoals meldingen over ontbrekende authenticatie, snelheidslimieten of overbelasting, kunnen nog steeds worden afgeleverd. Directe chats tonen standaard een compacte foutmelding; onbewerkte runnerdetails worden alleen weergegeven wanneer `/verbose full` is ingeschakeld.

Antwoorden die uitsluitend uit een stil token bestaan, worden op alle oppervlakken verwijderd, zodat bovenliggende sessies stil blijven in plaats van sentineltekst te herschrijven naar terugvalgebabbel.

## Gerelateerd

- [Refactor van de berichtlevenscyclus](/nl/concepts/message-lifecycle-refactor) - beoogd duurzaam ontwerp voor verzenden en ontvangen
- [Streaming](/nl/concepts/streaming) - realtime aflevering van berichten
- [Opnieuw proberen](/nl/concepts/retry) - gedrag voor het opnieuw proberen van berichtaflevering
- [Wachtrij](/nl/concepts/queue) - wachtrij voor berichtverwerking
- [Kanalen](/nl/channels) - integraties met berichtenplatforms
