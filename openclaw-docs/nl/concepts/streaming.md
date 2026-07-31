---
read_when:
    - Uitleg over hoe streaming of opdelen in chunks werkt op kanalen
    - Gedrag voor blokstreaming of kanaalopsplitsing wijzigen
    - Foutopsporing van dubbele/vroege blokantwoorden of streaming van kanaalvoorbeelden
summary: Streaming- en chunkinggedrag (blokantwoorden, streaming van kanaalvoorbeelden, modustoewijzing)
title: Streamen en opdelen in segmenten
x-i18n:
    generated_at: "2026-07-27T04:58:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a498f2e490ae6f2ecdebba92f0b992f2e16d212eae6a437eb3a0ef8a59354e13
    source_path: concepts/streaming.md
    workflow: 16
---

OpenClaw heeft twee onafhankelijke streaminglagen en er is momenteel **geen echte
token-deltastreaming** naar kanaalberichten:

- **Blokstreaming (kanalen):** voltooide **blokken** verzenden terwijl de assistent
  schrijft. Dit zijn normale kanaalberichten, geen tokendelta's.
- **Voorvertoningsstreaming (Telegram/Discord/Slack/Matrix/Mattermost/MS Teams):**
  tijdens het genereren een tijdelijk **voorvertoningsbericht** bijwerken (verzenden + bewerken/toevoegen).

## Opstartstatus van de Control UI

Nadat `chat.send` een actieve uitvoering bevestigt, kan de Gateway een getypeerde,
globale opstartstatus verzenden voordat tekst van de assistent of toolactiviteit zichtbaar is. De
Control UI toont deze status naast de werkindicator, met fasen voor
werkruimtevoorbereiding, inrichting van de omgeving, contextvoorbereiding en
het opstarten van het model.

De eerste delta van de assistent of de start van een tool vervangt de opstartstatus voor
die uitvoering definitief. De goedkeuringsstatus krijgt voorrang terwijl een tool wacht op actie
van de operator. Het aanmaken van de worktree en de eerste verzending naar de cloud vinden plaats voordat een chatuitvoering
bestaat, waardoor de voortgang van hun RPC vóór de uitvoering niet als opstartstatus van de uitvoering wordt weergegeven;
de inrichting van de omgeving verschijnt hier alleen wanneer een actieve uitvoering een
teruggewonnen worker opnieuw inricht.

## Blokstreaming (kanaalberichten)

Blokstreaming verzendt uitvoer van de assistent in grove delen zodra deze beschikbaar komt.

```text
Modeluitvoer
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ chunker verzendt blokken naarmate de buffer groeit
       └─ (blockStreamingBreak=message_end)
            └─ chunker leegt de buffer bij message_end
                   └─ verzending naar kanaal (blokantwoorden)
```

- `text_delta/events`: modelstreamgebeurtenissen (kunnen schaars zijn bij modellen zonder streaming).
- `chunker`: `EmbeddedBlockChunker` past minimum-/maximumgrenzen en de voorkeur voor afbreekpunten toe.
- `channel send`: daadwerkelijk uitgaande berichten (blokantwoorden).

**Besturingselementen** (allemaal onder `agents.defaults`, tenzij anders vermeld):

| Sleutel                                                      | Waarden / vorm                                                           | Standaardwaarde |
| ------------------------------------------------------------ | ------------------------------------------------------------------------ | --------------- |
| `blockStreamingDefault`                                      | `"on"` / `"off"`                                                        | `"off"`    |
| `blockStreamingBreak`                                        | `"text_end"` / `"message_end"`                                          | -          |
| `blockStreamingChunk`                                        | `{ minChars, maxChars, breakPreference? }`                              | -          |
| `blockStreamingCoalesce`                                     | `{ minChars?, maxChars?, idleMs? }` (gestreamde blokken vóór verzending samenvoegen) | -          |
| `*.streaming.block.enabled` (overschrijving per kanaal)               | `true` / `false`, dwingt blokstreaming af per kanaal (en per account)  | -          |
| `*.textChunkLimit` (bijv. `channels.whatsapp.textChunkLimit`) | getal, harde limiet                                                        | 4000       |
| `*.streaming.chunkMode`                                      | `"length"` / `"newline"`                                                | `"length"` |
| `channels.discord.maxLinesPerMessage`                        | getal, zachte regellimiet die lange antwoorden splitst om afkapping in de UI te voorkomen | 17         |

`streaming.chunkMode: "newline"` splitst op lege regels (alineagrenzen),
niet op elke nieuwe regel, voordat wordt teruggevallen op splitsing op lengte zodra de tekst
de limiet overschrijdt.

Gebundelde kanalen schrijven deze overschrijvingen als
`channels.<id>.streaming.{chunkMode,block.enabled,block.coalesce}`. De platte schrijfwijzen
`*.chunkMode` / `*.blockStreaming` / `*.blockStreamingCoalesce` zijn
verouderd voor elk gebundeld kanaal: `openclaw doctor --fix` migreert ze naar
de geneste vorm en kanaalschema's wijzen ze af. Configuraties van externe SDK-plugins
die nog steeds de platte schrijfwijzen gebruiken, blijven via een verouderde
terugvaloptie werken (met een runtimewaarschuwing) tot de volgende releasecyclus.

**Grenssemantiek** voor `blockStreamingBreak`:

- `text_end`: stream blokken zodra de chunker ze verzendt; leeg de buffer bij elke `text_end`.
- `message_end`: wacht tot het assistentbericht is voltooid en leeg vervolgens de gebufferde
  uitvoer. Gebruikt nog steeds de chunker als de gebufferde tekst `maxChars` overschrijdt, zodat deze
  aan het einde meerdere delen kan verzenden.

### Medialevering met blokstreaming

Streamingmedia moeten gestructureerde payloadvelden gebruiken, zoals `mediaUrl` of
`mediaUrls`; gestreamde tekst wordt niet als een bijlageopdracht geïnterpreteerd. Wanneer blokstreaming
media vroeg verzendt, onthoudt OpenClaw die levering voor de beurt. Als
de uiteindelijke payload van de assistent dezelfde media-URL herhaalt, verwijdert de uiteindelijke levering
de dubbele media in plaats van de bijlage opnieuw te verzenden.

Exact identieke uiteindelijke payloads worden onderdrukt. Als de uiteindelijke payload
afzonderlijke tekst toevoegt rond media die al zijn gestreamd, verzendt OpenClaw
de nieuwe tekst nog steeds, terwijl de media slechts eenmaal worden geleverd. Dit voorkomt dubbele spraakberichten
of bestanden op kanalen zoals Telegram.

## Splitsingsalgoritme (onder-/bovengrenzen)

Bloksplitsing wordt geïmplementeerd door `EmbeddedBlockChunker`:

- **Ondergrens:** niets verzenden totdat de buffer >= `minChars` is (tenzij afgedwongen).
- **Bovengrens:** geef de voorkeur aan splitsingen vóór `maxChars`; indien afgedwongen, splits bij `maxChars`.
- **Voorkeursvolgorde voor afbreekpunten:** `paragraph` -> `newline` -> `sentence` ->
  witruimte -> harde afbreking.
- **Codeblokken:** splits nooit binnen fences; sluit bij een afgedwongen splitsing op `maxChars`
  de fence en open deze opnieuw om geldige Markdown te behouden.

`maxChars` wordt begrensd op de `textChunkLimit` van het kanaal, zodat je
de limieten per kanaal niet kunt overschrijden.

## Samenvoegen (gestreamde blokken combineren)

Wanneer blokstreaming is ingeschakeld, kan OpenClaw **opeenvolgende blokdelen
samenvoegen** voordat ze worden verzonden. Dit vermindert spam van afzonderlijke regels terwijl
de uitvoer nog steeds geleidelijk wordt weergegeven.

- Bij samenvoegen wordt vóór het legen gewacht op **perioden van inactiviteit** (`idleMs`).
- Buffers worden begrensd door `maxChars` en geleegd als ze deze waarde overschrijden.
- `minChars` voorkomt dat kleine fragmenten worden verzonden totdat voldoende tekst is verzameld
  (bij de uiteindelijke leging wordt resterende tekst altijd verzonden).
- Het scheidingsteken wordt afgeleid van `blockStreamingChunk.breakPreference`: `paragraph` ->
  `\n\n`, `newline` -> `\n`, `sentence` -> spatie.
- Overschrijvingen per kanaal zijn beschikbaar via `*.streaming.block.coalesce` (inclusief
  configuraties per account).
- Discord, Signal en Slack gebruiken standaard `{ minChars: 1500, idleMs: 1000 }`
  voor samenvoegen, tenzij dit wordt overschreven.

## Menselijk aandoende pauzes tussen blokken

Voeg wanneer blokstreaming is ingeschakeld na het eerste blok een **willekeurige pauze**
tussen blokantwoorden toe, zodat antwoorden met meerdere tekstballonnen natuurlijker aanvoelen.

| `agents.defaults.humanDelay.mode` | Gedrag                  |
| --------------------------------- | ----------------------- |
| `off` (standaard)                   | Geen pauze              |
| `natural`                         | Willekeurige pauze van 800-2500ms |
| `custom`                          | `minMs`/`maxMs`         |

Overschrijf dit per agent via `agents.entries.*.humanDelay`. Is alleen van toepassing op **blokantwoorden**,
niet op uiteindelijke antwoorden of toolsamenvattingen.

## "Delen of alles streamen"

- **Delen streamen:** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"`
  (verzend gaandeweg). Voor andere kanalen dan Telegram is ook
  `*.streaming.block.enabled: true` vereist.
- **Alles aan het einde streamen:** `blockStreamingBreak: "message_end"` (buffer
  eenmaal legen, mogelijk in meerdere delen indien zeer lang).
- **Geen blokstreaming:** `blockStreamingDefault: "off"` (alleen het uiteindelijke antwoord).

Blokstreaming is **uitgeschakeld tenzij** `*.streaming.block.enabled` expliciet
is ingesteld op `true` (uitzondering: QQ Bot heeft geen `streaming.block`-sleutels en streamt
blokantwoorden tenzij `channels.qqbot.streaming.mode` gelijk is aan `"off"`). Kanalen kunnen
een livevoorvertoning streamen (`channels.<channel>.streaming.mode`) zonder blokantwoorden.
De standaardwaarden van `blockStreaming*` staan onder `agents.defaults`, niet in de
configuratiehoofdmap.

## Modi voor voorvertoningsstreaming

Canonieke sleutel: `channels.<channel>.streaming` (geneste `{ mode, ... }`; verouderde
booleaanse/tekenreeksvarianten op het hoogste niveau worden herschreven door `openclaw doctor --fix`).

| Modus      | Gedrag                                                                |
| ---------- | --------------------------------------------------------------------- |
| `off`      | Voorvertoningsstreaming uitschakelen                                  |
| `partial`  | Eén voorvertoning die door de nieuwste tekst wordt vervangen          |
| `block`    | Voorvertoning wordt in gesplitste/toegevoegde stappen bijgewerkt      |
| `progress` | Voortgangs-/statusvoorvertoning tijdens het genereren, definitief antwoord bij voltooiing |

`streaming.mode: "block"` is een modus voor voorvertoningsstreaming voor kanalen
die bewerken ondersteunen, zoals Discord en Telegram; deze modus schakelt daar niet zelfstandig
de levering van kanaalblokken in. Gebruik `streaming.block.enabled` voor normale blokantwoorden.
Microsoft Teams is de
uitzondering: het heeft geen bloktransport voor conceptvoorvertoningen, dus `streaming.mode:
"block"` schakelt native streaming volledig uit en het antwoord wordt als normale
bloklevering geplaatst in plaats van als native gedeeltelijke/voortgangsstreaming. Mattermost
wijkt ook af: in de modus `block` wisselt de voorvertoning tussen voltooide tekst en
toolactiviteitsblokken, zodat eerdere blokken als afzonderlijke berichten zichtbaar blijven
in plaats van te worden overschreven in één bewerkbaar concept.

### Kanaaltoewijzing

| Kanaal     | `off` | `partial` | `block` | `progress`              |
| ---------- | ----- | --------- | ------- | ----------------------- |
| Telegram   | Ja    | Ja        | Ja      | bewerkbaar voortgangsconcept |
| Discord    | Ja    | Ja        | Ja      | bewerkbaar voortgangsconcept |
| Slack      | Ja    | Ja        | Ja      | Ja                      |
| Mattermost | Ja    | Ja        | Ja      | Ja                      |
| MS Teams   | Ja    | Ja        | Ja      | native voortgangsstream |

De configuratie voor voorvertoningsdelen (`streaming.preview.chunk.*`, bijvoorbeeld onder
`channels.discord.streaming` of `channels.telegram.streaming`) gebruikt standaard
`minChars: 200`, `maxChars: 800` (begrensd op de `textChunkLimit` van het kanaal) en
`breakPreference: "paragraph"`.

Alleen voor Slack:

- `channels.slack.streaming.nativeTransport` schakelt aanroepen naar de native streaming-API van Slack
  (`chat.startStream`/`chat.appendStream`/`chat.stopStream`) in of uit wanneer
  `channels.slack.streaming.mode="partial"` (standaard: `true`).
- Voor native streaming en de threadstatus van de Slack-assistent is een antwoordthread
  als doel vereist. DM's op het hoogste niveau tonen die voorvertoning in threadstijl niet, maar kunnen
  nog steeds conceptvoorvertoningsberichten en bewerkingen van Slack gebruiken.

### Migratie van verouderde sleutels

| Kanaal   | Verouderde sleutels                                         | Status                                                                                                                                               |
| -------- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Telegram | `streamMode`, scalaire/booleaanse `streaming`                    | Door `openclaw doctor --fix` herschreven naar `streaming.mode`; niet gelezen tijdens runtime                                                                        |
| Discord  | `streamMode`, booleaanse `streaming`                           | Door `openclaw doctor --fix` herschreven naar `streaming.mode`; niet gelezen tijdens runtime                                                                        |
| Slack    | `streamMode`; booleaanse `streaming`; verouderde `nativeStreaming` | Door `openclaw doctor --fix` herschreven naar `streaming.mode` (en `streaming.nativeTransport` voor de booleaanse/verouderde vormen); niet gelezen tijdens runtime         |
| Matrix   | scalaire/booleaanse `streaming`                                  | Door `openclaw doctor --fix` herschreven naar `streaming.mode` (inclusief de `"quiet"`-modus van Matrix); niet gelezen tijdens runtime                                    |
| Feishu   | booleaanse `streaming`                                         | Door `openclaw doctor --fix` herschreven naar `streaming.mode`; niet gelezen tijdens runtime                                                                        |
| QQ Bot   | booleaanse `streaming`; `streaming.c2cStreamApi`               | Door `openclaw doctor --fix` herschreven naar `streaming.mode` (en `streaming.nativeTransport` voor de booleaanse/`c2cStreamApi`-vormen); niet gelezen tijdens runtime |

## Runtimegedrag

### Telegram

- Gebruikt `sendMessage` + `editMessageText`-voorbeeldupdates in privéberichten en
  groepen/onderwerpen; de definitieve tekst bewerkt het actieve voorbeeld ter plaatse. Tijdelijke
  Telegram-concepten voor 30 seconden met de status 'typen' (`sendMessageDraft`) worden niet gebruikt voor
  het streamen van antwoorden.
- Korte eerste voorbeelden worden nog steeds met debounce verwerkt voor de gebruikerservaring van pushmeldingen, maar
  verschijnen na een begrensde vertraging, zodat actieve uitvoeringen niet visueel stil blijven.
- Lange definitieve antwoorden hergebruiken het voorbeeldbericht voor het eerste fragment en verzenden alleen de
  resterende fragmenten.
- De `block`-modus zet het voorbeeld om in een nieuw bericht bij
  `streaming.preview.chunk.maxChars` (standaard 800, begrensd op de bewerkingslimiet van Telegram van 4096);
  andere modi laten één voorbeeld groeien tot 4096 tekens.
- De `progress`-modus houdt de voortgang van tools bij in een bewerkbaar statusconcept, toont
  het statuslabel wanneer antwoordstreaming actief is maar er nog geen toolregel
  beschikbaar is, wist het concept bij voltooiing en verzendt het definitieve antwoord
  via de normale bezorging.
- Als de definitieve bewerking mislukt voordat de voltooide tekst is bevestigd, gebruikt OpenClaw
  de normale definitieve bezorging en ruimt het verouderde voorbeeld op.
- Voorbeeldstreaming wordt overgeslagen wanneer Telegram-blokstreaming expliciet
  is ingeschakeld, om dubbele streaming te voorkomen.
- `/reasoning stream` kan redeneringen naar een tijdelijk voorbeeld schrijven dat
  na de definitieve bezorging wordt verwijderd.
- Geselecteerde citaatantwoorden van Telegram vormen een uitzondering: wanneer `replyToMode` niet
  `"off"` is en geselecteerde citaattekst aanwezig is, slaat OpenClaw de antwoordvoorbeeldstream
  voor die beurt over (het definitieve antwoord moet via het systeemeigen pad voor
  citaatantwoorden verlopen), zodat voorbeeldregels voor toolvoortgang niet kunnen worden weergegeven. Antwoorden op het huidige bericht
  zonder geselecteerde citaattekst behouden de voorbeeldstreaming. Zie
  [documentatie voor het Telegram-kanaal](/nl/channels/telegram) voor details.

### Discord

- Gebruikt verzonden en bewerkte voorbeeldberichten.
- De `block`-modus gebruikt conceptfragmentatie (`draftChunk`).
- Voorbeeldstreaming wordt overgeslagen wanneer Discord-blokstreaming expliciet
  is ingeschakeld.
- De `progress`-modus voegt een klein `-#`-activiteitenoverzicht (aantallen gedachten/toolaanroepen
  en verstreken tijd) toe aan het definitieve antwoord en verwijdert het statusconcept
  zodra dat antwoord is bezorgd, zodat drukke kanalen geen verweesd toollogboek
  boven het antwoord behouden. Bij definitieve foutberichten blijft het concept bewaard als registratie van de mislukte
  beurt.
- Definitieve media-, fout- en expliciete antwoordpayloads annuleren openstaande voorbeelden
  zonder een nieuw concept te publiceren en gebruiken vervolgens de normale bezorging.

### Slack

- `partial` kan, indien beschikbaar, systeemeigen Slack-streaming gebruiken (`chat.startStream`/`append`/`stop`).
- `block` gebruikt conceptvoorbeelden met toevoegingen.
- `progress` gebruikt statustekst als voorbeeld, gevolgd door het definitieve antwoord.
- Privéberichten op het hoogste niveau zonder antwoordthread gebruiken conceptvoorbeeldberichten en bewerkingen
  in plaats van systeemeigen Slack-streaming.
- Systeemeigen streaming en conceptvoorbeeldstreaming onderdrukken blokantwoorden voor die beurt, zodat een
  Slack-antwoord slechts via één bezorgingspad wordt gestreamd.
- Definitieve media-/foutpayloads en definitieve voortgangsberichten maken geen tijdelijke conceptberichten;
  alleen definitieve tekst-/blokberichten die het voorbeeld kunnen bewerken, publiceren openstaande
  concepttekst.

### Mattermost

- In de `partial`-modus worden gedachten en gedeeltelijke antwoordtekst gestreamd naar één
  conceptvoorbeeldbericht dat ter plaatse definitief wordt gemaakt wanneer het definitieve antwoord veilig kan worden verzonden.
- In de `progress`-modus worden gedachten en toolactiviteit gestreamd naar één statusvoorbeeld
  dat ter plaatse definitief wordt gemaakt wanneer het definitieve antwoord veilig kan worden verzonden.
- In de `block`-modus wordt gewisseld tussen berichten met voltooide tekst en toolactiviteit;
  parallelle en opeenvolgende toolupdates delen het huidige toolactiviteitsbericht.
- Valt terug op het verzenden van een nieuw definitief bericht als het voorbeeldbericht is verwijderd of
  anderszins niet beschikbaar is op het moment van definitief maken.
- Definitieve media-/foutpayloads annuleren openstaande voorbeeldupdates vóór de normale
  bezorging, in plaats van een tijdelijk voorbeeldbericht te publiceren.

### Matrix

- Conceptvoorbeelden worden ter plaatse definitief gemaakt wanneer de definitieve tekst de voorbeeldgebeurtenis
  kan hergebruiken.
- Definitieve berichten met alleen media, fouten en een niet-overeenkomend antwoorddoel annuleren openstaande voorbeeldupdates
  vóór de normale bezorging; een reeds zichtbaar verouderd voorbeeld wordt geredigeerd.

## Voorbeeldupdates voor toolvoortgang

Voorbeeldstreaming kan ook **toolvoortgangsupdates** bevatten: korte statusregels
zoals 'zoeken op internet', 'bestand lezen' of 'tool aanroepen' die
in hetzelfde voorbeeldbericht verschijnen terwijl tools worden uitgevoerd, vóór het definitieve antwoord.
In de Codex-appservermodus gebruiken Codex-inleidings-/commentaarberichten hetzelfde
voorbeeldpad, zodat korte voortgangsmeldingen zoals 'Ik controleer...' naar het
bewerkbare concept kunnen worden gestreamd zonder onderdeel van het definitieve antwoord te worden. Hierdoor blijven
toolbeurten met meerdere stappen visueel actief in plaats van stil tussen het eerste
voorbeeld van de gedachten en het definitieve antwoord.

Langdurige tools kunnen getypeerde voortgang uitsturen voordat ze terugkeren. Zo
start `web_fetch` bij aanvang een timer van vijf seconden: als het ophalen nog steeds
niet is voltooid, toont het voorbeeld `Fetching page content...`; als het ophalen voordien wordt voltooid of
geannuleerd, wordt geen voortgangsregel uitgestuurd. Het latere definitieve toolresultaat
wordt nog steeds normaal aan het model geleverd.

Ondersteunde oppervlakken:

- **Discord**, **Slack**, **Telegram** en **Matrix** streamen toolvoortgangs- en
  Codex-inleidingsupdates standaard naar de live voorbeeldbewerking wanneer voorbeeldstreaming
  actief is. Microsoft Teams gebruikt zijn systeemeigen voortgangsstream in
  persoonlijke chats.
- Telegram wordt sinds `v2026.4.22` geleverd met ingeschakelde toolvoortgangsupdates
  voor voorbeelden; door deze ingeschakeld te houden, blijft dat uitgebrachte gedrag behouden.
- **Mattermost** voegt toolactiviteit samen in één voorbeeldbericht in de modi `partial` en
  `progress`, of in één toolactiviteitsbericht tussen tekstblokken in de `block`-modus
  (zie hierboven).
- Toolvoortgangsbewerkingen volgen de actieve voorbeeldstreamingmodus; ze worden
  overgeslagen wanneer voorbeeldstreaming `off` is of wanneer blokstreaming het
  bericht heeft overgenomen. Op Telegram is `streaming.mode: "off"` alleen definitief: algemeen
  voortgangscommentaar wordt eveneens onderdrukt in plaats van als afzonderlijke statusberichten
  bezorgd, terwijl goedkeuringsprompts, mediapayloads en fouten nog steeds
  normaal worden gerouteerd.
- Om voorbeeldstreaming te behouden maar toolvoortgangsregels te verbergen, stel je
  `streaming.preview.toolProgress` voor dat kanaal in op `false` (standaard
  `true`). Om toolvoortgangsregels zichtbaar te houden terwijl opdracht-/uitvoertekst wordt verborgen,
  stel je `streaming.preview.commandText` in op `"status"` of
  `streaming.progress.commandText` op `"status"`; de standaardwaarde is `"raw"` om
  uitgebracht gedrag te behouden. Dit beleid wordt gedeeld door concept-/voortgangskanalen
  die de compacte voortgangsrenderer van OpenClaw gebruiken, waaronder Discord, Matrix,
  Microsoft Teams, Mattermost, Slack-conceptvoorbeelden en Telegram. Om
  voorbeeldbewerkingen volledig uit te schakelen, stel je `streaming.mode` in op `off`.

## Weergave van voortgangsconcepten

Voortgangsconcepten (`streaming.progress.*`) zijn begrensd en per
kanaal configureerbaar:

| Sleutel                           | Standaard     | Gedrag                                                         |
| --------------------------------- | ------------- | -------------------------------------------------------------- |
| `streaming.progress.maxLines`     | `8`           | Maximaal aantal compacte voortgangsregels onder het conceptlabel |
| `streaming.progress.maxLineChars` | `120`         | Maximaal aantal tekens per compacte regel vóór afkapping (woordbewust) |
| `streaming.progress.label`        | `"auto"`      | Concepttitel; een aangepaste tekenreeks, of `false` om deze te verbergen |
| `streaming.progress.labels`       | ingebouwde verzameling | Kandidaatlabels die worden gebruikt wanneer `label: "auto"` |

### Voortgangsbaan voor commentaar

Naast toolvoortgang kan de compacte voortgangsrenderer nog één baan
in het concept weergeven:

- **`streaming.progress.commentary`** - geef het **commentaar** van het model vóór toolgebruik
  weer (een korte beschrijving zoals 'Ik controleer... en daarna...'), afgewisseld met
  toolregels in het voortgangsconcept. Op Discord en Telegram in de voortgangsmodus
  levert dezelfde inleiding de statuskop, zelfs wanneer deze optionele baan
  is uitgeschakeld; andere kanalen behouden hun bestaande voortgangsgedrag. Zie
  [Voortgangsconcepten](/nl/concepts/progress-drafts#status-headline).

```json
{
  "channels": {
    "discord": {
      "streaming": { "mode": "progress", "progress": { "commentary": true } }
    }
  }
}
```

Houd voortgangsregels zichtbaar, maar verberg onbewerkte opdracht-/uitvoertekst:

```json
{
  "channels": {
    "telegram": {
      "streaming": {
        "mode": "partial",
        "preview": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

Gebruik dezelfde structuur onder de sleutel van een ander compact voortgangskanaal, bijvoorbeeld
`channels.discord`, `channels.matrix`, `channels.msteams`,
`channels.mattermost` of Slack-conceptvoorbeelden. Plaats voor de voortgangsconceptmodus
hetzelfde beleid onder `streaming.progress`:

```json
{
  "channels": {
    "telegram": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

## Gerelateerd

- [Refactor van de berichtlevenscyclus](/nl/concepts/message-lifecycle-refactor) - ontwerp voor gedeelde voorbeelden, bewerkingen, streams en definitieve verwerking
- [Voortgangsconcepten](/nl/concepts/progress-drafts) - zichtbare berichten over werk in uitvoering die tijdens lange beurten worden bijgewerkt
- [Berichten](/nl/concepts/messages) - berichtlevenscyclus en bezorging
- [Opnieuw proberen](/nl/concepts/retry) - gedrag voor opnieuw proberen bij een bezorgingsfout
- [Kanalen](/nl/channels) - ondersteuning voor streaming per kanaal
