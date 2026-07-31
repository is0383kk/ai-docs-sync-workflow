---
read_when:
    - Je wilt zoeken op het web zonder een API-sleutel
    - Je wilt de betaalde Search API van Parallel
    - Je wilt compacte fragmenten die zijn gerangschikt op efficiëntie voor LLM-context
summary: Parallel zoeken -- voor LLM's geoptimaliseerde compacte fragmenten uit webbronnen
title: Parallel zoeken
x-i18n:
    generated_at: "2026-07-27T06:16:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eff693f286015b287bbdacf44f11ff6f07f2f7d2605ef6f09259e7402b40515e
    source_path: tools/parallel-search.md
    workflow: 16
---

De Parallel-Plugin biedt twee [Parallel](https://parallel.ai/) `web_search`
providers, die beide gerangschikte, voor LLM's geoptimaliseerde fragmenten retourneren uit een webindex
die voor AI-agents is gebouwd:

| Provider               | id              | Authenticatie                                                                                       |
| ---------------------- | --------------- | ------------------------------------------------------------------------------------------ |
| Parallel Search (gratis) | `parallel-free` | Geen -- Parallels gratis [Search MCP](https://docs.parallel.ai/integrations/mcp/search-mcp) |
| Parallel Search        | `parallel`      | `PARALLEL_API_KEY` -- betaalde Search API, hogere frequentielimieten en afstemming op doelstellingen             |

Stel `tools.web.search.provider` in op `parallel-free` of `parallel` om
er expliciet één te selecteren; geen van beide wordt automatisch gedetecteerd.

<Note>
  Directe OpenAI Responses-modellen (`api: "openai-responses"`, provider
  `openai`, officiële API-basis-URL) gebruiken automatisch OpenAI's gehoste native webzoekfunctie
  wanneer `tools.web.search.provider` niet is ingesteld, leeg is, `"auto"`
  of `"openai"` is -- standaard omzeilen ze Parallel dus. Stel
  `tools.web.search.provider` in op `parallel-free` of `parallel` om ze
  in plaats daarvan via Parallel te routeren. Zie [Overzicht van webzoeken](/nl/tools/web).
</Note>

## Plugin installeren

```bash
openclaw plugins install @openclaw/parallel-plugin
openclaw gateway restart
```

## API-sleutel (betaalde provider)

`parallel-free` heeft geen sleutel nodig, maar moet nog steeds expliciet worden geselecteerd. De betaalde
provider `parallel` heeft een API-sleutel nodig:

<Steps>
  <Step title="Een account aanmaken">
    Meld je aan bij [platform.parallel.ai](https://platform.parallel.ai) en
    genereer een API-sleutel vanuit je dashboard.
  </Step>
  <Step title="De sleutel opslaan">
    Stel `PARALLEL_API_KEY` in de Gateway-omgeving in, of configureer deze via:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## Configuratie

```json5
{
  plugins: {
    entries: {
      parallel: {
        config: {
          webSearch: {
            apiKey: "par-...", // optioneel als PARALLEL_API_KEY is ingesteld
            baseUrl: "https://api.parallel.ai", // optioneel; OpenClaw voegt /v1/search toe
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        // "parallel-free" voor de gratis Search MCP, of "parallel" voor de
        // hier getoonde betaalde provider met API-backend.
        provider: "parallel",
      },
    },
  },
}
```

**Alternatief via de omgeving:** stel `PARALLEL_API_KEY` in de Gateway-
omgeving in. Plaats deze voor een Gateway-installatie in `~/.openclaw/.env`.

## Basis-URL overschrijven

Geldt alleen voor de betaalde provider `parallel`; `parallel-free` gebruikt altijd
`https://search.parallel.ai/mcp` en negeert deze instelling.

Stel `plugins.entries.parallel.config.webSearch.baseUrl` in om betaalde
verzoeken via een compatibele proxy of alternatief eindpunt te routeren (bijvoorbeeld de
Cloudflare AI Gateway). OpenClaw normaliseert kale hosts door
`https://` ervoor te plaatsen en voegt `/v1/search` toe, tenzij het pad daar al op eindigt. Het
opgeloste eindpunt maakt deel uit van de cachecode voor zoekopdrachten, zodat resultaten van verschillende
eindpunten nooit worden gedeeld.

## Toolparameters

Beide providers bieden de native zoekstructuur van Parallel, zodat het model een
doel in natuurlijke taal plus enkele korte trefwoordzoekopdrachten invult -- de combinatie
die Parallel [aanbeveelt](https://docs.parallel.ai/search/best-practices) voor
de beste resultaten.

<ParamField path="objective" type="string" required>
Beschrijving in natuurlijke taal van de onderliggende vraag of het doel (max. 5000
tekens). Moet op zichzelf staan.
</ParamField>

<ParamField path="search_queries" type="string[]" required>
Beknopte zoekopdrachten met trefwoorden, elk 3-6 woorden (1-5 vermeldingen, max. 200 tekens
per stuk). Geef voor de beste resultaten 2-3 gevarieerde zoekopdrachten op.
</ParamField>

<ParamField path="count" type="number">
Aantal te retourneren resultaten (1-40).
</ParamField>

<ParamField path="session_id" type="string">
Optionele Parallel-sessie-id uit `sessionId` van een eerder resultaat. Geef deze door bij
vervolgzoekopdrachten binnen dezelfde taak, zodat Parallel gerelateerde aanroepen groepeert en
volgende resultaten verbetert. Max. 1000 tekens voor `parallel`; de gratis
`parallel-free` Search MCP beperkt dit tot 100. Een id die de limiet overschrijdt, wordt verwijderd
(betaald) of er wordt een nieuwe aangemaakt (gratis).
</ParamField>

<ParamField path="client_model" type="string">
Optionele identificatie van het model dat de aanroep uitvoert (bijv. `claude-opus-4-7`,
`gpt-5.6-sol`), max. 100 tekens. Hiermee kan Parallel de standaardinstellingen afstemmen op de
mogelijkheden van je model. Geef de exacte slug van het actieve model door; kort deze niet in tot een
familiealias.
</ParamField>

## Opmerkingen

- Parallel rangschikt en comprimeert resultaten voor gebruik bij redeneringen door LLM's, niet voor
  doorklikken door mensen; verwacht compacte fragmenten per resultaat in plaats van volledige
  pagina-inhoud.
- Resultaatfragmenten worden geretourneerd als de array `excerpts` en worden ook samengevoegd in
  `description` voor compatibiliteit met het algemene contract `web_search`.
- Beide providers retourneren een `session_id`; OpenClaw maakt deze als `sessionId` beschikbaar in
  de toolpayload, zodat aanroepers vervolgzoekopdrachten kunnen groeperen. Een
  door Parallel gegenereerde sessie-id (die niet door de aanroeper is opgegeven) wordt uitgesloten
  van de cachevermelding, omdat niet-gerelateerde taken met identieke zoekopdrachten
  deze niet mogen overnemen.
- `searchId`, `warnings` en `usage` van Parallel worden ongewijzigd doorgegeven wanneer
  ze aanwezig zijn.
- OpenClaw stuurt altijd een vastgesteld aantal resultaten door naar Parallel als
  `advanced_settings.max_results` (`parallel`) of past `count`
  aan de clientzijde toe na Parallels respons met vaste grootte (`parallel-free`). Het
  argument `count` van de aanroeper heeft voorrang, daarna `tools.web.search.maxResults`, en anders geldt
  OpenClaws algemene standaardwaarde `web_search` (5) -- de eigen API van Parallel gebruikt standaard
  10.
- Resultaten worden standaard 15 minuten in de cache opgeslagen (`cacheTtlMinutes`).
- `parallel-free` maakt per aanroep via de MCP-handshake een nieuwe `session_id` aan
  wanneer de aanroeper er geen opgeeft; `parallel` laat deze in dat
  geval oningesteld.

## Gerelateerd

- [Overzicht van webzoeken](/nl/tools/web) -- alle providers en automatische detectie
- [Exa-zoekfunctie](/nl/tools/exa-search) -- neuraal zoeken met inhoudsextractie
- [Perplexity Search](/nl/tools/perplexity-search) -- gestructureerde resultaten met domeinfiltering
