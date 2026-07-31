---
read_when:
    - Je wilt begrijpen wat 'context' betekent in OpenClaw
    - Je onderzoekt waarom het model iets 'weet' (of is vergeten)
    - Je wilt de contextoverhead verminderen (/context, /status, /compact)
summary: 'Context: wat het model ziet, hoe die wordt opgebouwd en hoe je die inspecteert'
title: Context
x-i18n:
    generated_at: "2026-07-27T05:48:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1eb3d342a601a447487640587f746cc80a133ede338a880741f53c3e01f20ed1
    source_path: concepts/context.md
    workflow: 16
---

"Context" is **alles wat OpenClaw voor een uitvoering naar het model stuurt**. Dit wordt begrensd door het **contextvenster** van het model (tokenlimiet).

Eenvoudig mentaal model voor beginners:

- **Systeemprompt** (opgebouwd door OpenClaw): regels, tools, lijst met Skills, tijd/runtime en geïnjecteerde werkruimtebestanden.
- **Gespreksgeschiedenis**: jouw berichten + de berichten van de assistent voor deze sessie.
- **Toolaanroepen/-resultaten + bijlagen**: opdrachtuitvoer, gelezen bestanden, afbeeldingen/audio, enzovoort.

Context is _niet hetzelfde_ als "geheugen": geheugen kan op schijf worden opgeslagen en later opnieuw worden geladen; context is wat zich in het huidige venster van het model bevindt.

## Snel aan de slag (context inspecteren)

- `/status` → snelle weergave van "hoe vol is mijn venster?" + sessie-instellingen.
- `/context list` → wat wordt geïnjecteerd + geschatte grootten (per bestand + totalen).
- `/context detail` → gedetailleerdere uitsplitsing: grootten per bestand, per toolschema en per Skill-vermelding, grootte van de systeemprompt en aantallen comprimeerbare transcriptberichten.
- `/context map` → treemapafbeelding in WinDirStat-stijl van de bijgehouden contextbijdragen van de huidige sessie.
- `/usage tokens` → voegt aan normale antwoorden een voettekst toe met het gebruik per antwoord.
- `/compact` → vat oudere geschiedenis samen in een compacte vermelding om ruimte in het venster vrij te maken.

Zie ook: [Slash-opdrachten](/nl/tools/slash-commands), [Tokengebruik en -kosten](/nl/reference/token-use), [Compaction](/nl/concepts/compaction).

## Voorbeelduitvoer

Waarden verschillen per model, provider, toolbeleid en wat zich in je werkruimte bevindt.

### `/context list`

```text
🧠 Contextuitsplitsing
Werkruimte: <workspaceDir>
Bootstrapmaximum/bestand: 12,000 tekens
Sandbox: modus=non-main in sandbox=false
Systeemprompt (uitvoering): 38,412 tekens (~9,603 tok) (Projectcontext 23,901 tekens (~5,976 tok))

Geïnjecteerde werkruimtebestanden:
- AGENTS.md: OK | onbewerkt 1,742 tekens (~436 tok) | geïnjecteerd 1,742 tekens (~436 tok)
- SOUL.md: OK | onbewerkt 912 tekens (~228 tok) | geïnjecteerd 912 tekens (~228 tok)
- TOOLS.md: AFGEKAPT | onbewerkt 54,210 tekens (~13,553 tok) | geïnjecteerd 20,962 tekens (~5,241 tok)
- IDENTITY.md: OK | onbewerkt 211 tekens (~53 tok) | geïnjecteerd 211 tekens (~53 tok)
- USER.md: OK | onbewerkt 388 tekens (~97 tok) | geïnjecteerd 388 tekens (~97 tok)
- HEARTBEAT.md: ONTBREEKT | onbewerkt 0 | geïnjecteerd 0
- BOOTSTRAP.md: OK | onbewerkt 0 tekens (~0 tok) | geïnjecteerd 0 tekens (~0 tok)

Lijst met Skills (tekst van systeemprompt): 2,184 tekens (~546 tok) (12 Skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Lijst met tools (tekst van systeemprompt): 1,032 tekens (~258 tok)
Toolschema's (JSON): 31,988 tekens (~7,997 tok) (tellen mee voor de context; niet als tekst weergegeven)
Tools: (zelfde als hierboven)

Sessietokens (in cache): 14,250 totaal / ctx=32,000
```

### `/context detail`

```text
🧠 Contextuitsplitsing (gedetailleerd)
…
Grootste Skills (grootte van promptvermelding):
- frontend-design: 412 tekens (~103 tok)
- oracle: 401 tekens (~101 tok)
… (+10 meer Skills)

Grootste tools (schemagrootte):
- browser: 9,812 tekens (~2,453 tok)
- exec: 6,240 tekens (~1,560 tok)
… (+N meer tools)
```

### `/context map`

Verstuurt een afbeelding die is gegenereerd uit het meest recente uitvoeringsrapport in de cache en het sessietranscript. Voordat een normaal bericht in de sessie een uitvoeringsrapport heeft geproduceerd, retourneert `/context map` een melding dat dit niet beschikbaar is in plaats van een schatting weer te geven. De oppervlakte van de rechthoeken is evenredig aan het aantal bijgehouden prompttekens:

- gesprekstranscript (gebruikersberichten, antwoorden van de assistent, toolresultaten, Compaction-samenvattingen), plus runtimecontext per beurt en aanvullingen op de hookprompt die alleen het model bereiken
- geïnjecteerde werkruimtebestanden
- basistekst van de systeemprompt
- Skill-promptvermeldingen
- JSON-schema's van tools

De gespreksgroep groeit mee met de sessie, waardoor de kaart elke beurt verandert; na Compaction wordt deze samengevouwen tot een tegel met samenvattingen.

`/context list`, `/context detail` en `/context json` kunnen nog steeds een schatting op aanvraag inspecteren wanneer er geen uitvoeringsrapport in de cache staat.

## Wat meetelt voor het contextvenster

Alles wat het model ontvangt, telt mee, waaronder:

- Systeemprompt (alle secties).
- Gespreksgeschiedenis.
- Toolaanroepen + toolresultaten.
- Bijlagen/transcripten (afbeeldingen/audio/bestanden).
- Compaction-samenvattingen en opschoningsartefacten.
- Provider-"wrappers" of verborgen headers (niet zichtbaar, maar tellen wel mee).

## Hoe OpenClaw de systeemprompt opbouwt

De systeemprompt is **eigendom van OpenClaw** en wordt voor elke uitvoering opnieuw opgebouwd. Deze bevat:

- Lijst met tools + korte beschrijvingen.
- Lijst met Skills (alleen metadata; zie hieronder).
- Locatie van de werkruimte.
- Tijd (UTC + omgerekende gebruikerstijd indien geconfigureerd).
- Runtimemetadata (host/OS/model/denkmodus).
- Geïnjecteerde bootstrapbestanden van de werkruimte onder **Projectcontext**.

Volledige uitsplitsing: [Systeemprompt](/nl/concepts/system-prompt).

## Geïnjecteerde werkruimtebestanden (Projectcontext)

OpenClaw injecteert standaard een vaste reeks werkruimtebestanden (indien aanwezig):

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (alleen bij de eerste uitvoering)

Grote bestanden worden per bestand afgekapt met `agents.defaults.bootstrapMaxChars` (standaard `20000` tekens). OpenClaw hanteert daarnaast met `agents.defaults.bootstrapTotalMaxChars` een totale limiet voor bootstrapinjectie over alle bestanden heen (standaard `60000` tekens). `/context` toont de **onbewerkte versus geïnjecteerde** grootten en of er afkapping heeft plaatsgevonden.

Wanneer afkapping plaatsvindt, kan de runtime onder Projectcontext een waarschuwingsblok in de prompt injecteren. Configureer dit met `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`; standaard `always`).

## Skills: geïnjecteerd versus op aanvraag geladen

De systeemprompt bevat een compacte **lijst met Skills** (naam + beschrijving + locatie). Deze lijst zorgt daadwerkelijk voor extra belasting.

Skill-instructies worden standaard _niet_ opgenomen. Van het model wordt verwacht dat het `read` het bestand `SKILL.md` van de Skill **alleen wanneer nodig**.

## Tools: er zijn twee kostenposten

Tools beïnvloeden de context op twee manieren:

1. **Tekst van de lijst met tools** in de systeemprompt (wat je ziet als "Tooling").
2. **Toolschema's** (JSON). Deze worden naar het model gestuurd zodat het tools kan aanroepen. Ze tellen mee voor de context, ook al zie je ze niet als platte tekst.

`/context detail` splitst de grootste toolschema's uit, zodat je kunt zien wat het zwaarst weegt.

## Opdrachten, richtlijnen en "inline snelkoppelingen"

Slash-opdrachten worden verwerkt door de Gateway. Er zijn enkele verschillende gedragingen:

- **Zelfstandige opdrachten**: een bericht dat alleen uit `/...` bestaat, wordt als opdracht uitgevoerd.
- **Richtlijnen**: `/think`, `/fast`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue` worden verwijderd voordat het model het bericht ziet.
  - Berichten die alleen richtlijnen bevatten, bewaren de sessie-instellingen.
  - Inline richtlijnen in een normaal bericht fungeren als aanwijzingen per bericht.
- **Inline snelkoppelingen** (alleen afzenders op de toelatingslijst): bepaalde `/...`-tokens in een normaal bericht kunnen onmiddellijk worden uitgevoerd (voorbeeld: "hé /status") en worden verwijderd voordat het model de resterende tekst ziet.

Details: [Slash-opdrachten](/nl/tools/slash-commands).

## Sessies, Compaction en opschoning (wat behouden blijft)

Wat tussen berichten behouden blijft, hangt af van het mechanisme:

- **Normale geschiedenis** blijft in het sessietranscript staan totdat deze volgens beleid wordt gecomprimeerd/opgeschoond.
- **Compaction** slaat een samenvatting op in het transcript en houdt recente berichten intact.
- **Opschoning** verwijdert oude toolresultaten uit de prompt _in het geheugen_ om ruimte in het contextvenster vrij te maken, maar herschrijft het sessietranscript niet - de volledige geschiedenis blijft op schijf inspecteerbaar.

Documentatie: [Sessie](/nl/concepts/session), [Compaction](/nl/concepts/compaction), [Sessies opschonen](/nl/concepts/session-pruning).

OpenClaw gebruikt standaard de ingebouwde `legacy`-contextengine voor samenstelling en
Compaction. Als je een Plugin installeert die `kind: "context-engine"` levert en
deze selecteert met `plugins.slots.contextEngine`, delegeert OpenClaw de samenstelling van de context,
`/compact` en gerelateerde levenscyclus-hooks voor subagentcontext aan die
engine. `ownsCompaction: false` valt niet automatisch terug op de verouderde
engine; de actieve engine moet `compact()` nog steeds correct implementeren. Zie
[Contextengine](/nl/concepts/context-engine) voor de volledige
uitbreidbare interface, levenscyclus-hooks en configuratie.

## Wat `/context` daadwerkelijk rapporteert

`/context` geeft waar mogelijk de voorkeur aan het meest recente **tijdens de uitvoering opgebouwde** rapport van de systeemprompt:

- `System prompt (run)` = vastgelegd tijdens de laatste ingebedde uitvoering (met toolmogelijkheden) en bewaard in de sessieopslag.
- `System prompt (estimate)` = direct berekend wanneer er geen uitvoeringsrapport bestaat (of bij uitvoering via een CLI-backend die het rapport niet genereert).

In beide gevallen rapporteert het grootten en de grootste bijdragen; het dumpt **niet** de volledige systeemprompt of toolschema's. In de gedetailleerde modus vergelijkt het ook het sessietranscript met hetzelfde predicaat voor echte gespreksberichten dat door Compaction wordt gebruikt, zodat hoog prompt-/cachegebruik eenvoudiger te onderscheiden is van comprimeerbare gespreksgeschiedenis.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Contextengine" href="/nl/concepts/context-engine" icon="puzzle-piece">
    Aangepaste contextinjectie via plugins.
  </Card>
  <Card title="Compaction" href="/nl/concepts/compaction" icon="compress">
    Lange gesprekken samenvatten om ze binnen het modelvenster te houden.
  </Card>
  <Card title="Systeemprompt" href="/nl/concepts/system-prompt" icon="message-lines">
    Hoe de systeemprompt wordt opgebouwd en wat deze bij elke beurt injecteert.
  </Card>
  <Card title="Agentlus" href="/nl/concepts/agent-loop" icon="arrows-rotate">
    De volledige uitvoeringscyclus van de agent, van binnenkomend bericht tot definitief antwoord.
  </Card>
</CardGroup>
