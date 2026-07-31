---
read_when:
    - Je wilt dat OpenClaw-agents een grote toolcatalogus gebruiken zonder elk toolschema aan de prompt toe te voegen
    - Je wilt OpenClaw-tools, MCP-tools en clienttools beschikbaar maken via één compact runtime-oppervlak
    - Je implementeert of debugt tooldetectie voor OpenClaw-runs
summary: 'Tool Search: maak grote OpenClaw-toolcatalogi compact met zoeken, beschrijven en aanroepen'
title: Zoeken naar tools
x-i18n:
    generated_at: "2026-07-27T05:55:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d31322d5ef108c52fd14d48771cc3c6c43fcfbc4bfb95652bc29a55fd706c903
    source_path: tools/tool-search.md
    workflow: 16
---

Tool Search is een experimentele functie van de OpenClaw-agentruntime. Hiermee kunnen agents op één
compacte manier grote toolcatalogi doorzoeken en aanroepen. Dit is nuttig wanneer voor de uitvoering
veel tools beschikbaar zijn, maar het model er waarschijnlijk slechts enkele nodig heeft.

Deze pagina documenteert OpenClaw Tool Search. Dit is niet de Codex-eigen interface voor het
zoeken naar tools of dynamische tools. De Codex-eigen codemodus, het zoeken naar tools, uitgestelde
dynamische tools en geneste toolaanroepen zijn stabiele interfaces van de Codex-harness en zijn
niet afhankelijk van `tools.toolSearch`.

Zie [Codemodus](/tools/code-mode) voor de algemene OpenClaw-runtime die een QuickJS-WASI `exec`/`wait`-
interface beschikbaar stelt in plaats van Tool Search-besturingselementen.

Wanneer dit voor OpenClaw-uitvoeringen is ingeschakeld, ontvangt het model standaard één `tool_search_code`-tool,
plus alle tools die alleen rechtstreeks beschikbaar zijn en waarvan de gestructureerde resultaten niet via
de compacte bridge kunnen worden doorgegeven. De codetool voert een korte JavaScript-body uit in een geïsoleerd
Node-subproces met een `openclaw.tools`-bridge:

```js
const hits = await openclaw.tools.search("een GitHub-issue aanmaken");
const tool = await openclaw.tools.describe(hits[0].id);
return await openclaw.tools.call(tool.id, {
  title: "Crash bij opstarten",
  body: "Stappen om te reproduceren...",
});
```

De catalogus kan voor de catalogus geschikte OpenClaw-tools, plugintools, MCP-
tools en door de client aangeleverde tools bevatten. Het model ziet niet vooraf elk gecatalogiseerd schema.
In plaats daarvan doorzoekt het compacte descriptoren, vraagt het een beschrijving van één geselecteerde
tool op wanneer het exacte schema nodig is, en roept het die tool via OpenClaw aan.
Tools die alleen rechtstreeks beschikbaar zijn, blijven zichtbaar voor het model en worden niet aan de catalogus toegevoegd.

Uitvoeringen met de Codex-harness ontvangen deze experimentele OpenClaw Tool Search-
besturingselementen niet. OpenClaw geeft productmogelijkheden als dynamische tools door aan Codex, en
Codex beheert de stabiele eigen codemodus, het eigen zoeken naar tools, uitgestelde dynamische
tools en geneste toolaanroepen.

## Hoe een beurt wordt uitgevoerd

Tijdens de planning bouwt de ingebedde OpenClaw-runner de effectieve catalogus voor de
uitvoering:

1. Bepaal het actieve toolbeleid voor de agent, het profiel, de sandbox en de sessie.
2. Geef de geschikte OpenClaw- en plugintools weer.
3. Geef de geschikte MCP-tools weer via de MCP-runtime van de sessie.
4. Voeg geschikte clienttools toe die voor de huidige uitvoering zijn aangeleverd.
5. Houd tools die alleen rechtstreeks beschikbaar zijn zichtbaar voor het model en indexeer compacte descriptoren voor de
   overige voor de catalogus geschikte tools.
6. Stel de OpenClaw-codebridge, de gestructureerde fallbacktools of de
   compacte directory-interface beschikbaar naast die tools die alleen rechtstreeks beschikbaar zijn.

Tijdens de uitvoering keert elke daadwerkelijke toolaanroep terug naar OpenClaw. De geïsoleerde Node-
runtime bevat geen pluginimplementaties, MCP-clientobjecten of geheimen.
`openclaw.tools.call(...)` gaat via de bridge terug naar de Gateway, waar het
normale beleid en de normale goedkeuring, hooks, logging en resultaatverwerking nog steeds van toepassing zijn.

## Modi

`tools.toolSearch` heeft drie voor het model zichtbare modi:

- `code`: stelt `tool_search_code`, de standaard compacte JavaScript-bridge, beschikbaar
  naast tools die alleen rechtstreeks beschikbaar zijn.
- `tools`: stelt `tool_search`, `tool_describe` en `tool_call` beschikbaar als gewone
  gestructureerde tools voor providers die geen code mogen ontvangen, naast
  tools die alleen rechtstreeks beschikbaar zijn.
- `directory`: stelt `tool_search`, `tool_describe` en `tool_call` beschikbaar, plus een
  begrensde promptdirectory met namen en beschrijvingen van beschikbare tools voor
  providers die toolnamen moeten zien zonder elk volledig schema. OpenClaw kan
  voor de huidige beurt ook rechtstreeks een kleine, begrensde set waarschijnlijke of vereiste toolschema's
  beschikbaar stellen. Tools die alleen rechtstreeks beschikbaar zijn, blijven ook in deze modus zichtbaar.

Alle modi gebruiken dezelfde door beleid gefilterde catalogus en het normale OpenClaw-uitvoeringspad.
Tools die zijn gemarkeerd als `catalogMode: "direct-only"` blijven buiten die catalogus en
blijven zichtbaar voor het model. Als de huidige runtime het geïsoleerde Node-kindproces voor de codemodus
niet kan starten, valt de standaardmodus `code` vóór cataloguscompactie terug op `tools`.
In de modus `directory` blijven door de client aangeleverde tools voor de huidige uitvoering
rechtstreeks zichtbaar, terwijl OpenClaw-tools, plugintools en MCP-tools achter
de directorycatalogus kunnen worden gecompacteerd. Een rechtstreekse aanroep van een exacte verborgen
directorynaam wordt vóór uitvoering vanuit diezelfde geautoriseerde catalogus geladen.

Alle modi zijn experimenteel. Geef bij kleine OpenClaw-toolcatalogi de voorkeur aan rechtstreekse
toolbeschikbaarheid en geef bij uitvoeringen met de Codex-harness de voorkeur aan de stabiele Codex-eigen interfaces.

Er is geen afzonderlijke configuratie voor bronselectie. Wanneer Tool Search is ingeschakeld, bevat de
catalogus na normale beleidsfiltering de voor de catalogus geschikte OpenClaw-, MCP- en clienttools;
tools die alleen rechtstreeks beschikbaar zijn, worden afzonderlijk behouden.

## Waarom dit bestaat

Grote catalogi zijn nuttig, maar kostbaar. Als elk toolschema naar het model wordt verzonden,
wordt de aanvraag groter, verloopt de planning trager en neemt de kans op onbedoelde toolselectie toe.

Tool Search verandert de vorm:

- rechtstreekse tools: het model ziet elk geselecteerd schema vóór het eerste token
- Tool Search-codemodus: het model ziet één compacte codetool, een kort API-
  contract en alle tools die alleen rechtstreeks beschikbaar zijn
- Tool Search-toolsmodus: het model ziet drie compacte gestructureerde fallbacktools,
  plus alle tools die alleen rechtstreeks beschikbaar zijn
- Tool Search-directorymodus: het model ziet een begrensde directory plus
  besturingselementen voor zoeken/beschrijven/aanroepen en een kleine, begrensde set waarschijnlijke of vereiste
  schema's, plus alle tools die alleen rechtstreeks beschikbaar zijn
- tijdens de beurt: het model kan naar behoefte de overige schema's laden

Rechtstreekse toolbeschikbaarheid blijft de juiste standaard voor kleine catalogi. Tool Search
werkt het beste wanneer één uitvoering veel tools kan zien, met name van MCP-servers of
door de client aangeleverde apptools.

## API

`openclaw.tools.search(query, options?)`

Doorzoekt de effectieve catalogus voor de huidige uitvoering. Resultaten zijn compact en kunnen veilig
weer in de promptcontext worden opgenomen. Elke treffer bevat een begrensde TypeScript-achtige
`input`-signatuur, zoals `{ id: string; mode?: "drip" | "flood" }`, zodat het
model `describe` kan overslaan wanneer die signatuur volstaat. Een vertrouwde
OpenClaw-core- of plugintool kan ook een compacte `output`-hint bevatten, zoals
`Array<{ id: string; paid: boolean }>`. Claims over uitvoerschema's van MCP en clients worden
niet in deze vertrouwde hint opgenomen. Hun niet-vertrouwde invoerschema's worden ook
uitgesteld als `input: "unknown"`; gebruik `describe` voordat je ze aanroept. Open,
te grote of anderszins gedeeltelijke uitvoerschema's laten de hint weg en blijven in plaats daarvan
beschikbaar via `describe`.

```js
const hits = await openclaw.tools.search("agenda-afspraak", { limit: 5 });
```

`openclaw.tools.describe(id)`

Laadt volledige metadata voor één zoekresultaat, inclusief het exacte invoerschema en
de vertrouwde volledige `outputSchema` wanneer de tool er een declareert.

```js
const calendarCreate = await openclaw.tools.describe("mcp:calendar:create_event");
```

`openclaw.tools.call(id, args)`

Roept een geselecteerde tool aan via OpenClaw en retourneert de onbewerkte `{ tool, result }`-
envelop. Tools die JSON retourneren, plaatsen hun waarde normaal gesproken in
`result.details`. Als een vertrouwde tool `outputSchema` declareert, compileert OpenClaw
het schema vóór uitvoering en valideert het de uiteindelijke `details` na de normale tool-
hooks voordat de catalogusaanroep wordt geretourneerd.

```js
await openclaw.tools.call(calendarCreate.id, {
  summary: "Planning",
  start: "2026-05-09T14:00:00Z",
});
```

Toolauteurs declareren uitvoercontracten in de eigenschap `outputSchema` van de tool.
Deze beschrijft `AgentToolResult.details`, niet gerenderde inhoudsblokken. Neem
alle varianten op die geen uitzondering veroorzaken, of laat de eigenschap weg bij instabiele resultaten. Zie
[Uitvoercontracten voor de codemodus](/tools/code-mode#declared-output-contracts) en
[Toolplugins](/nl/plugins/tool-plugins#output-contracts).

De gestructureerde fallbackmodus stelt dezelfde bewerkingen als tools beschikbaar:

- `tool_search`
- `tool_describe`
- `tool_call`

De directorymodus stelt het volgende beschikbaar:

- `tool_search`
- `tool_describe`
- `tool_call`

Deze modus houdt ook door de client aangeleverde tools en alle tools die alleen rechtstreeks beschikbaar zijn direct zichtbaar,
en kan voor de huidige beurt rechtstreeks een kleine, begrensde set waarschijnlijke of vereiste
catalogustoolschema's beschikbaar stellen. Als de begrensde directory vermeldingen weglaat, gebruik je
`tool_search` om ze te vinden. Als het model rechtstreeks een exacte verborgen directory-
toolnaam aanvraagt, laadt OpenClaw deze vóór de normale uitvoering uit de geautoriseerde catalogus.
Namen van clienttools in de directorymodus mogen niet conflicteren met namen van OpenClaw-, plugin- of MCP-
tools, omdat exacte uitgestelde dispatch deze namen gebruikt.

## Runtimegrens

De codebridge wordt uitgevoerd in een kortdurend Node-subproces. Het subproces start
met de Node-machtigingsmodus ingeschakeld, een lege omgeving, zonder toestemmingen voor het bestandssysteem of
netwerk en zonder toestemmingen voor kindprocessen of workers. OpenClaw dwingt vanuit het
bovenliggende proces een time-out op basis van verstreken kloktijd af en beëindigt het subproces bij een time-out, ook
na asynchrone vervolgstappen.

De runtime stelt alleen het volgende beschikbaar:

- `console.log`, `console.warn` en `console.error`
- `openclaw.tools.search`
- `openclaw.tools.describe`
- `openclaw.tools.call`

Het normale OpenClaw-gedrag blijft van toepassing op uiteindelijke aanroepen:

- beleid voor het toestaan en weigeren van tools
- toolbeperkingen per agent en per sandbox
- toolbeleid voor kanaal/runtime
- goedkeuringshooks
- pluginhooks voor `before_tool_call`
- sessie-identiteit, logboeken en telemetrie

## Configuratie

Schakel Tool Search voor OpenClaw-uitvoeringen in met de standaardcodebridge:

```bash
openclaw config set tools.toolSearch true
```

Gelijkwaardige JSON:

```json5
{
  tools: {
    toolSearch: true,
  },
}
```

Gebruik in plaats daarvan de gestructureerde fallbacktools voor OpenClaw-uitvoeringen:

```json5
{
  tools: {
    toolSearch: {
      mode: "tools",
    },
  },
}
```

Gebruik in plaats daarvan de compacte directory-interface voor OpenClaw-uitvoeringen:

```json5
{
  tools: {
    toolSearch: {
      mode: "directory",
    },
  },
}
```

Stel de time-out van de codemodus en de limieten voor zoekresultaten af (de getoonde waarden zijn de standaardwaarden):

```json5
{
  tools: {
    toolSearch: {
      mode: "code",
      codeTimeoutMs: 10000,
      searchDefaultLimit: 8,
      maxSearchLimit: 20,
    },
  },
}
```

De runtime begrenst `codeTimeoutMs` tot 1000-60000, `maxSearchLimit` tot 1-50 en
`searchDefaultLimit` tot 1..`maxSearchLimit`.

Schakel de functie uit:

```json5
{
  tools: {
    toolSearch: false,
  },
}
```

## Prompt en telemetrie

Tool Search registreert voldoende telemetrie om het met rechtstreekse toolbeschikbaarheid te vergelijken:

- totaal aantal geserialiseerde tool- en promptbytes dat naar de harness is verzonden
- catalogusgrootte en uitsplitsing per bron
- aantallen zoek-, beschrijf- en aanroepbewerkingen
- uiteindelijke toolaanroepen die via OpenClaw zijn uitgevoerd
- geselecteerde tool-id's en bronnen

Met sessielogboeken moet het mogelijk zijn om het volgende te bepalen:

- hoeveel toolschema's het model vooraf zag
- hoeveel zoek- en beschrijfbewerkingen het uitvoerde
- welke uiteindelijke tool werd aangeroepen
- of het resultaat afkomstig was van OpenClaw, MCP of een clienttool

## E2E-validatie

Het Gateway-scenario van QA Lab bewijst beide paden met de OpenClaw-runtime:

```bash
pnpm openclaw qa suite --provider-mode mock-openai --scenario tool-search-gateway-e2e
```

Het maakt een tijdelijke nepplugin met een grote toolcatalogus, start de nagebootste
OpenAI-provider, start een Gateway eenmaal in de rechtstreekse modus en eenmaal met Tool Search
ingeschakeld, en vergelijkt vervolgens de aanvraagpayloads van de provider en de sessielogboeken.

De regressietest bewijst:

1. De directe modus kan de tool van de nepplugin aanroepen.
2. Tool Search kan dezelfde tool van de nepplugin aanroepen.
3. De directe modus stelt de schema's van de tool van de nepplugin rechtstreeks beschikbaar aan de provider.
4. Tool Search stelt alleen de compacte bridge en eventuele tools die uitsluitend rechtstreeks beschikbaar zijn ter beschikking.
5. De aanvraagpayload van Tool Search is kleiner voor de grote nepcatalogus.
6. Sessielogboeken tonen de verwachte aantallen toolaanroepen en telemetrie voor aanroepen via de bridge.

## Gedrag bij fouten

Tool Search moet standaard blokkeren:

- als een tool niet in het effectieve beleid staat, mag de zoekopdracht deze niet retourneren
- als een geselecteerde tool niet meer beschikbaar is, moet `tool_call` mislukken
- als beleid of goedkeuring de uitvoering blokkeert, moet het aanroepresultaat die
  blokkering melden in plaats van deze te omzeilen
- als de codebridge geen geïsoleerde runtime kan maken, gebruik dan `mode: "tools"` of
  schakel Tool Search uit voor die implementatie

## Gerelateerd

- [Tools en plugins](/nl/tools)
- [Multi-agent-sandbox en -tools](/nl/tools/multi-agent-sandbox-tools)
- [Exec-tool](/nl/tools/exec)
- [Configuratie van ACP-agents](/nl/tools/acp-agents-setup)
- [Plugins bouwen](/nl/plugins/building-plugins)
