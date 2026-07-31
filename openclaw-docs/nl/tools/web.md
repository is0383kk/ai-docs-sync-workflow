---
read_when:
    - Je wilt web_search inschakelen of configureren
    - Je wilt x_search inschakelen of configureren
    - Je moet een zoekprovider kiezen
    - Je wilt automatische detectie en providerselectie begrijpen
sidebarTitle: Web Search
summary: web_search, x_search en web_fetch -- doorzoek het web, doorzoek berichten op X of haal pagina-inhoud op
title: Zoeken op het web
x-i18n:
    generated_at: "2026-07-27T05:29:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 997e51064b0cd08d0f30987aa038e2f4a98da22f1094974b45f59c18491bd979
    source_path: tools/web.md
    workflow: 16
---

`web_search` doorzoekt het web met je geconfigureerde provider en retourneert
genormaliseerde resultaten, die per zoekopdracht 15 minuten in de cache worden bewaard (configureerbaar). OpenClaw
bevat ook `x_search` voor berichten op X (voorheen Twitter) en `web_fetch` voor
het lichtgewicht ophalen van URL's. `web_fetch` wordt altijd lokaal uitgevoerd; `web_search` wordt
via xAI Responses gerouteerd wanneer Grok de provider is, en `x_search` gebruikt altijd
xAI Responses.

<Info>
  `web_search` is een lichtgewicht HTTP-tool, geen browserautomatisering. Gebruik voor
  sites die sterk afhankelijk zijn van JS of voor aanmeldingen de [webbrowser](/nl/tools/browser). Gebruik voor
  het ophalen van een specifieke URL [Web Fetch](/nl/tools/web-fetch).
</Info>

## Snel aan de slag

<Steps>
  <Step title="Kies een provider">
    Kies een provider en voltooi alle vereiste configuratie. Sommige providers werken
    zonder sleutel, andere vereisen een API-sleutel. Raadpleeg de onderstaande providerpagina's voor
    meer informatie.
  </Step>
  <Step title="Configureren">
    ```bash
    openclaw configure --section web
    ```
    Hiermee worden de provider en eventuele benodigde referenties opgeslagen. Voor providers
    met een API kun je in plaats daarvan de omgevingsvariabele van de provider instellen (bijvoorbeeld
    `BRAVE_API_KEY`) en deze stap overslaan.
  </Step>
  <Step title="Gebruiken">
    ```javascript
    await web_search({ query: "OpenClaw plugin SDK" });
    ```

    Voor berichten op X:

    ```javascript
    await x_search({ query: "dinner recipes" });
    ```

  </Step>
</Steps>

## Een provider kiezen

<CardGroup cols={2}>
  <Card title="Brave Search" icon="shield" href="/nl/tools/brave-search">
    Gestructureerde resultaten met fragmenten. Ondersteunt de modus `llm-context` en land-/taalfilters. Gratis abonnement beschikbaar.
  </Card>
  <Card title="Codex Hosted Search" icon="search" href="/nl/plugins/codex-harness">
    Door AI samengestelde, op bronnen gebaseerde antwoorden via je Codex-appserveraccount.
  </Card>
  <Card title="DuckDuckGo" icon="bird" href="/nl/tools/duckduckgo-search">
    Provider zonder sleutel. Geen API-sleutel nodig. Onofficiële integratie op basis van HTML.
  </Card>
  <Card title="Exa" icon="brain" href="/nl/tools/exa-search">
    Neuraal zoeken en zoeken op trefwoorden met inhoudsextractie (markeringen, tekst, samenvattingen).
  </Card>
  <Card title="Firecrawl" icon="flame" href="/nl/tools/firecrawl">
    Gestructureerde resultaten. Werkt het beste in combinatie met `firecrawl_search` en `firecrawl_scrape` voor diepgaande extractie.
  </Card>
  <Card title="Gemini" icon="sparkles" href="/nl/tools/gemini-search">
    Door AI samengestelde antwoorden met bronvermeldingen via onderbouwing door Google Zoeken.
  </Card>
  <Card title="Grok" icon="zap" href="/nl/tools/grok-search">
    Door AI samengestelde antwoorden met bronvermeldingen via webonderbouwing van xAI.
  </Card>
  <Card title="Kimi" icon="moon" href="/nl/tools/kimi-search">
    Door AI samengestelde antwoorden met bronvermeldingen via de webzoekfunctie van Moonshot; niet-onderbouwde terugvallen op chat mislukken expliciet.
  </Card>
  <Card title="MiniMax Search" icon="globe" href="/nl/tools/minimax-search">
    Gestructureerde resultaten via de zoek-API van het MiniMax Token Plan.
  </Card>
  <Card title="Ollama Web Search" icon="globe" href="/nl/tools/ollama-search">
    Zoeken via een aangemelde lokale Ollama-host of de gehoste Ollama-API.
  </Card>
  <Card title="Parallel" icon="layer-group" href="/nl/tools/parallel-search">
    Betaalde Parallel Search-API (`PARALLEL_API_KEY`); hogere frequentielimieten en afstemming op doelstellingen.
  </Card>
  <Card title="Parallel Search (gratis)" icon="layer-group" href="/nl/tools/parallel-search">
    Optioneel en zonder sleutel. De gratis Search MCP van Parallel, met compacte, voor LLM's geoptimaliseerde fragmenten en zonder API-sleutel.
  </Card>
  <Card title="Perplexity" icon="search" href="/nl/tools/perplexity-search">
    Gestructureerde resultaten met instellingen voor inhoudsextractie en domeinfiltering.
  </Card>
  <Card title="SearXNG" icon="server" href="/nl/tools/searxng-search">
    Zelfgehost metazoeken. Geen API-sleutel nodig. Combineert Google, Bing, DuckDuckGo en meer.
  </Card>
  <Card title="Tavily" icon="globe" href="/nl/tools/tavily">
    Gestructureerde resultaten met zoekdiepte, onderwerpfiltering en `tavily_extract` voor URL-extractie.
  </Card>
</CardGroup>

### Providers vergelijken

| Provider                                         | Resultaatstijl                                                   | Filters                                          | API-sleutel                                                                                 |
| ------------------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| [Brave](/nl/tools/brave-search)                     | Gestructureerde fragmenten                                            | Land, taal, tijd, modus `llm-context`      | `BRAVE_API_KEY`                                                                         |
| [Codex Hosted Search](/nl/plugins/codex-harness)    | Door AI samengesteld + bron-URL's                                   | Domeinen, contextgrootte, gebruikerslocatie             | Geen; gebruikt aanmelding bij Codex/OpenAI                                                         |
| [DuckDuckGo](/nl/tools/duckduckgo-search)           | Gestructureerde fragmenten                                            | --                                               | Geen (zonder sleutel)                                                                         |
| [Exa](/nl/tools/exa-search)                         | Gestructureerd + geëxtraheerd                                         | Neurale/trefwoordmodus, datum, inhoudsextractie    | `EXA_API_KEY`                                                                           |
| [Firecrawl](/nl/tools/firecrawl)                    | Gestructureerde fragmenten                                            | Via de tool `firecrawl_search`                      | `FIRECRAWL_API_KEY`                                                                     |
| [Gemini](/nl/tools/gemini-search)                   | Door AI samengesteld + bronvermeldingen                                     | --                                               | `GEMINI_API_KEY`                                                                        |
| [Grok](/nl/tools/grok-search)                       | Door AI samengesteld + bronvermeldingen                                     | --                                               | xAI OAuth, `XAI_API_KEY` of `plugins.entries.xai.config.webSearch.apiKey`              |
| [Kimi](/nl/tools/kimi-search)                       | Door AI samengesteld + bronvermeldingen; mislukt bij niet-onderbouwde terugvallen op chat | --                                               | `KIMI_API_KEY` / `MOONSHOT_API_KEY`                                                     |
| [MiniMax Search](/nl/tools/minimax-search)          | Gestructureerde fragmenten                                            | Regio (`global` / `cn`)                         | `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN`              |
| [Ollama Web Search](/nl/tools/ollama-search)        | Gestructureerde fragmenten                                            | --                                               | Geen voor aangemelde lokale hosts; `OLLAMA_API_KEY` voor rechtstreeks zoeken met `https://ollama.com` |
| [Parallel](/nl/tools/parallel-search)               | Compacte fragmenten gerangschikt voor LLM-context                          | --                                               | `PARALLEL_API_KEY` (betaald)                                                               |
| [Parallel Search (gratis)](/nl/tools/parallel-search) | Compacte fragmenten gerangschikt voor LLM-context                          | --                                               | Geen (gratis Search MCP)                                                                  |
| [Perplexity](/nl/tools/perplexity-search)           | Gestructureerde fragmenten                                            | Land, taal, tijd, domeinen, inhoudslimieten | `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY`                                             |
| [SearXNG](/nl/tools/searxng-search)                 | Gestructureerde fragmenten                                            | Categorieën, taal                             | Geen (zelfgehost)                                                                      |
| [Tavily](/nl/tools/tavily)                          | Gestructureerde fragmenten                                            | Via de tool `tavily_search`                         | `TAVILY_API_KEY`                                                                        |

## Resultaatstructuur

`web_search` normaliseert elke ingebouwde en externe pluginprovider op de grens van de
kerntool. Aanroepers ontvangen precies één van deze gesloten structuren:

```typescript
type WebSearchOutput =
  | {
      kind: "error";
      provider: string;
      error: "provider_error";
      message: string;
      docs?: string;
    }
  | {
      kind: "results";
      provider: string;
      query: string;
      count: number;
      tookMs?: number;
      results: Array<{
        title: string;
        url: string;
        snippet?: string;
        published?: string;
        siteName?: string;
      }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "answer";
      provider: string;
      query: string;
      tookMs?: number;
      content: string;
      citations?: Array<{ url: string; title?: string }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "raw";
      provider: string;
      data: unknown;
    };
```

Gestructureerde providers gebruiken `kind: "results"`; providers met samengestelde antwoorden gebruiken
`kind: "answer"`. Externe pluginproviders waarvan de payloads met geen van beide structuren
overeenkomen, worden voor compatibiliteit ongewijzigd doorgegeven als `kind: "raw"`. Providerspecifieke
velden zoals ruwe scores, fragmenten, gerelateerde zoekopdrachten, offsets van
inline bronvermeldingen, model-ID's of sessiemetadata worden niet doorgegeven in genormaliseerde
vertakkingen. Gebruik de specifieke tool van een provider wanneer de uitgebreidere respons ervan deel
uitmaakt van je workflow.

`externalContent.wrapped: true` is een vertrouwensmarkering die door de grens zelf
waar wordt gemaakt: tekst van de provider (`title`, `snippet`, `siteName`, `content`, titels van
bronvermeldingen, `message` van fouten) wordt ontdaan van eventuele bestaande omhullingsregels en
precies één keer opnieuw omhuld op de kerngrens, zodat metagegevens van providers de
markering niet kunnen vervalsen. `query` is altijd de aangevraagde zoekopdracht, URL's van bronvermeldingen en resultaten
moeten als http(s) kunnen worden geparseerd, `published` moet de vorm van een ISO-datum hebben, URL's worden in gecanonicaliseerde vorm uitgevoerd en een
payload met een sleutel `error` wordt altijd gerapporteerd als `kind: "error"`, waarbij de
ruwe providercode behouden blijft in het omhulde bericht. Ongewijzigd doorgegeven
payloads behouden alle markeringen die de provider heeft ingesteld.

## Automatische detectie

Providerlijsten in documentatie en configuratiestromen staan in alfabetische volgorde. Automatische detectie gebruikt een
afzonderlijke, vaste prioriteitsvolgorde en kiest alleen een provider waarvoor
referenties (`requiresCredential !== false`) nodig zijn wanneer geconfigureerde referenties worden gevonden. Als
`provider` niet is ingesteld, controleert OpenClaw providers in deze volgorde en gebruikt het
de eerste die gereed is:

Eerst providers met een API:

1. **Brave** -- `BRAVE_API_KEY` of `plugins.entries.brave.config.webSearch.apiKey` (volgorde 10)
2. **MiniMax Search** -- `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN` / `MINIMAX_API_KEY` of `plugins.entries.minimax.config.webSearch.apiKey` (volgorde 15)
3. **Gemini** -- `plugins.entries.google.config.webSearch.apiKey`, `GEMINI_API_KEY` of `models.providers.google.apiKey` (volgorde 20)
4. **Grok** -- xAI OAuth, `XAI_API_KEY` of `plugins.entries.xai.config.webSearch.apiKey` (volgorde 30)
5. **Kimi** -- `KIMI_API_KEY` / `MOONSHOT_API_KEY` of `plugins.entries.moonshot.config.webSearch.apiKey` (volgorde 40)
6. **Perplexity** -- `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY` of `plugins.entries.perplexity.config.webSearch.apiKey` (volgorde 50)
7. **Firecrawl** -- `FIRECRAWL_API_KEY` of `plugins.entries.firecrawl.config.webSearch.apiKey` (volgorde 60)
8. **Exa** -- `EXA_API_KEY` of `plugins.entries.exa.config.webSearch.apiKey`; optioneel overschrijft `plugins.entries.exa.config.webSearch.baseUrl` het Exa-eindpunt (volgorde 65)
9. **Tavily** -- `TAVILY_API_KEY` of `plugins.entries.tavily.config.webSearch.apiKey` (volgorde 70)
10. **Parallel** -- betaalde Parallel Search API via `PARALLEL_API_KEY` of `plugins.entries.parallel.config.webSearch.apiKey`; optioneel overschrijft `plugins.entries.parallel.config.webSearch.baseUrl` het eindpunt (volgorde 75)

Daarna volgen geconfigureerde eindpuntproviders:

11. **SearXNG** -- `SEARXNG_BASE_URL` of `plugins.entries.searxng.config.webSearch.baseUrl` (volgorde 200)

Providers zonder sleutel, zoals **Parallel Search (Free)**, **DuckDuckGo**,
**Ollama Web Search** en **Codex Hosted Search**, krijgen nooit voorrang bij automatische detectie,
hoewel ze een interne volgordewaarde hebben. Ze worden alleen gebruikt wanneer je
ze expliciet selecteert met `tools.web.search.provider` of via
`openclaw configure --section web`. OpenClaw stuurt beheerde
`web_search`-query's niet naar een provider zonder sleutel alleen omdat er geen API-ondersteunde
provider is geconfigureerd.

OpenAI Responses-modellen vormen een uitzondering: zolang `tools.web.search.provider`
niet is ingesteld, gebruiken ze de systeemeigen webzoekfunctie van OpenAI in plaats van de beheerde
providers hierboven (zie hieronder). Stel `tools.web.search.provider` in op
`parallel-free` (of een andere provider) om ze in plaats daarvan via het beheerde pad
te routeren.

<Note>
  Alle providersleutelvelden ondersteunen SecretRef-objecten. Plugin-specifieke SecretRefs
  onder `plugins.entries.<plugin>.config.webSearch.apiKey` worden opgelost voor de
  geïnstalleerde API-ondersteunde webzoekproviders, waaronder Brave, Exa, Firecrawl,
  Gemini, Grok, Kimi, MiniMax, Parallel, Perplexity en Tavily,
  ongeacht of de provider expliciet wordt gekozen via `tools.web.search.provider` of
  via automatische detectie wordt geselecteerd. In de modus voor automatische detectie lost OpenClaw alleen de
  geselecteerde providersleutel op -- niet-geselecteerde SecretRefs blijven inactief, zodat je
  meerdere providers geconfigureerd kunt houden zonder resolutiekosten te betalen voor
  de providers die je niet gebruikt.
</Note>

## Systeemeigen OpenAI-webzoekfunctie

Directe OpenAI Responses-modellen (`api: "openai-responses"`, provider `openai`,
geen basis-URL of een officiële OpenAI API-basis-URL) gebruiken automatisch OpenAI's gehoste
`web_search`-tool wanneer OpenClaw-webzoeken is ingeschakeld en geen
beheerde provider is vastgezet. Dit is gedrag dat eigendom is van de provider in de meegeleverde
OpenAI-plugin en is niet van toepassing op OpenAI-compatibele proxybasis-URL's of Azure-
routes. Stel `tools.web.search.provider` in op een andere provider, zoals `brave`, om
de beheerde `web_search`-tool voor OpenAI-modellen te blijven gebruiken, of stel
`tools.web.search.enabled: false` in om zowel beheerd zoeken als systeemeigen
OpenAI-zoeken uit te schakelen.

## Systeemeigen Codex-webzoekfunctie

De Codex-app-serverruntime gebruikt automatisch de gehoste `web_search`-tool van Codex
wanneer webzoeken is ingeschakeld en geen beheerde provider is geselecteerd. Systeemeigen gehost
zoeken en de dynamische beheerde `web_search`-tool van OpenClaw sluiten elkaar uit,
zodat beheerd zoeken de systeemeigen domeinbeperkingen niet kan omzeilen. OpenClaw gebruikt de
beheerde tool wanneer gehost zoeken niet beschikbaar of expliciet uitgeschakeld is, of
wordt vervangen door een geselecteerde beheerde provider. OpenClaw houdt de zelfstandige
`web.run`-extensie van Codex uitgeschakeld (`features.standalone_web_search: false`),
omdat productie-app-serververkeer de door de gebruiker gedefinieerde `web`-
naamruimte weigert.

- Configureer systeemeigen zoeken onder `tools.web.search.openaiCodex`
- Stel `tools.web.search.provider: "codex"` in om Codex Hosted Search beschikbaar te stellen als
  de beheerde `web_search`-provider voor elk bovenliggend model. Elke aanroep voert een
  begrensde, tijdelijke Codex-app-serverbeurt uit en mislukt als Codex geen
  gehost `webSearch`-item produceert.
- `mode: "cached"` is de standaardvoorkeur, maar Codex zet deze om in live
  externe toegang voor onbeperkte app-serverbeurten; stel `"live"` in om
  expliciet live toegang aan te vragen
- Stel `tools.web.search.provider` in op een beheerde provider, zoals `brave`, om
  in plaats daarvan de beheerde `web_search` van OpenClaw te gebruiken
- Stel `tools.web.search.openaiCodex.enabled: false` in om Codex-gehost
  zoeken uit te schakelen; andere beheerde providers blijven beschikbaar
- Door het systeemeigen Codex-tooloppervlak te beperken, blijft de beheerde `web_search`
  ook beschikbaar
- Wanneer `allowedDomains` is ingesteld, wordt automatische beheerde terugval gesloten afgebroken als
  gehost zoeken niet beschikbaar is, zodat de systeemeigen toelatingslijst niet kan worden omzeild
- LLM-only-uitvoeringen waarbij tools zijn uitgeschakeld, schakelen zowel systeemeigen als beheerd zoeken uit
- `tools.web.search.enabled: false` schakelt zowel beheerd als systeemeigen zoeken uit

Blijvende wijzigingen in het effectieve Codex-zoekbeleid starten een nieuwe gebonden thread, zodat
een reeds geladen app-serverthread geen verouderde toegang tot gehost zoeken kan behouden.
Tijdelijke beperkingen per beurt gebruiken een tijdelijke beperkte thread en behouden
de bestaande binding om later te hervatten.

Rechtstreeks OpenAI ChatGPT Responses-verkeer kan ook OpenAI's gehoste
`web_search`-tool gebruiken. Dat afzonderlijke pad blijft opt-in via
`tools.web.search.openaiCodex.enabled: true` en is alleen van toepassing op geschikte
`openai/*`-modellen die `api: "openai-chatgpt-responses"` gebruiken.

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        // Optioneel: gebruik Codex Hosted Search ook vanuit bovenliggende modellen die geen Codex zijn.
        provider: "codex",
        openaiCodex: {
          enabled: true,
          mode: "cached",
          allowedDomains: ["example.com"],
          contextSize: "high",
          userLocation: {
            country: "US",
            city: "New York",
            timezone: "America/New_York",
          },
        },
      },
    },
  },
}
```

Voor runtimes en providers die systeemeigen Codex-zoeken niet ondersteunen, kan Codex
de beheerde `web_search`-terugval gebruiken via de dynamische toolnaamruimte van OpenClaw.
Gebruik een expliciete beheerde provider wanneer je de providerspecifieke
netwerkcontroles van OpenClaw nodig hebt in plaats van door Codex gehost zoeken.

Door `provider: "codex"` te selecteren, wordt de meegeleverde `codex`-plugin ingeschakeld en worden
dezelfde hierboven getoonde `tools.web.search.openaiCodex`-beperkingen gebruikt. Verifieer eerst de identiteit van
de Codex-app-server met `openclaw models auth login --provider openai`.
De bovenliggende agent kan elk model of elke runtime gebruiken; alleen de begrensde zoekworker
wordt via Codex uitgevoerd.

## Netwerkveiligheid

Beheerde HTTP-aanroepen van de `web_search`-provider gebruiken het beveiligde ophaalpad van OpenClaw,
beperkt tot de eigen hostnaam van de huidige provider. Alleen voor die hostnaam
staat OpenClaw fake-IP-DNS-antwoorden van Surge, Clash en sing-box toe in
`198.18.0.0/15` en `fc00::/7`. Andere privé-, loopback-, link-local- en
metadatabestemmingen blijven geblokkeerd. Codex Hosted Search vormt de uitzondering:
de begrensde worker delegeert netwerktoegang aan de gehoste
`web_search`-tool van de Codex-app-server.

Deze automatische toestemming is niet van toepassing op willekeurige `web_fetch`-URL's. Schakel voor
`web_fetch` de opties `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` en
`tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` alleen expliciet in wanneer je
vertrouwde proxy eigenaar is van die synthetische bereiken.

## Configuratie

```json5
{
  tools: {
    web: {
      search: {
        enabled: true, // standaard: true
        provider: "brave", // of weglaten voor automatische detectie
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
    },
  },
}
```

Providerspecifieke configuratie (API-sleutels, basis-URL's, modi) staat onder
`plugins.entries.<plugin>.config.webSearch.*`. Gemini kan ook
`models.providers.google.apiKey` en `models.providers.google.baseUrl` hergebruiken als terugvalopties
met lagere prioriteit, na de specifieke webzoekconfiguratie en `GEMINI_API_KEY`. Zie de
providerpagina's voor voorbeelden.
Grok kan ook een xAI OAuth-authenticatieprofiel uit `openclaw models auth login
--provider xai --method oauth` hergebruiken; configuratie met een API-sleutel blijft de terugvaloptie.

`tools.web.search.provider` wordt gevalideerd aan de hand van de webzoekprovider-id's
die door meegeleverde en geïnstalleerde pluginmanifesten zijn gedeclareerd. Een typefout zoals `"brvae"`
zorgt ervoor dat de configuratievalidatie mislukt in plaats van stilzwijgend terug te vallen op automatische detectie. Als een
geconfigureerde provider alleen verouderd pluginbewijs heeft, zoals een achtergebleven
`plugins.entries.<plugin>`-blok na het verwijderen van een externe plugin,
blijft OpenClaw robuust opstarten en meldt het een waarschuwing, zodat je de
plugin opnieuw kunt installeren of `openclaw doctor --fix` kunt uitvoeren om de verouderde configuratie op te schonen.

De selectie van de `web_fetch`-terugvalprovider staat los hiervan:

- kies deze met `tools.web.fetch.provider`
- of laat dat veld weg en laat OpenClaw automatisch de eerste gereedstaande web-fetchprovider
  detecteren op basis van geconfigureerde referenties
- niet-gesandboxte `web_fetch` kan geïnstalleerde pluginproviders gebruiken die
  `contracts.webFetchProviders` declareren; gesandboxte ophaalacties staan meegeleverde providers en
  geverifieerde officiële plugininstallaties toe, maar sluiten externe plugins van derden uit
- de officiële Firecrawl-plugin is momenteel de enige meegeleverde bijdrager aan `webFetchProviders`,
  geconfigureerd onder
  `plugins.entries.firecrawl.config.webFetch.*`

Wanneer je **Kimi** kiest tijdens `openclaw onboard` of
`openclaw configure --section web`, kan OpenClaw ook vragen om:

- de Moonshot API-regio (`https://api.moonshot.ai/v1` of `https://api.moonshot.cn/v1`)
- het standaardmodel voor Kimi-webzoeken (standaard `kimi-k2.6`)

Configureer voor `x_search` de optie `plugins.entries.xai.config.xSearch.*`. Deze gebruikt hetzelfde
xAI-authenticatieprofiel als chat, of de `XAI_API_KEY`-referentie / pluginreferentie voor webzoeken
die door Grok-webzoeken wordt gebruikt.
Verouderde `tools.web.x_search.*`-configuratie wordt automatisch gemigreerd door `openclaw doctor --fix`.
Wanneer je Grok kiest tijdens `openclaw onboard` of `openclaw configure --section web`,
biedt OpenClaw ook optionele configuratie van `x_search` aan met dezelfde referentie, direct
nadat de Grok-configuratie is voltooid. Dit is een afzonderlijke vervolgstap binnen het Grok-
pad, geen afzonderlijke webzoekproviderkeuze op het hoogste niveau. Als je een andere
provider kiest, toont OpenClaw de `x_search`-prompt niet.

### API-sleutels opslaan

<Tabs>
  <Tab title="Configuratiebestand">
    Voer `openclaw configure --section web` uit of stel de sleutel rechtstreeks in:

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "YOUR_KEY", // pragma: allowlist secret
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="Omgevingsvariabele">
    Stel de omgevingsvariabele van de provider in de procesomgeving van de Gateway in:

    ```bash
    export BRAVE_API_KEY="YOUR_KEY"
    ```

    Plaats deze voor een Gateway-installatie in `~/.openclaw/.env`.
    Zie [Omgevingsvariabelen](/nl/help/faq#env-vars-and-env-loading).

  </Tab>
</Tabs>

## Toolparameters

| Parameter             | Beschrijving                                                        |
| --------------------- | ------------------------------------------------------------------ |
| `query`               | Zoekopdracht (verplicht)                                            |
| `count`               | Aantal te retourneren resultaten (1-10, standaard: 5)                               |
| `country`             | 2-letterige ISO-landcode (bijv. "US", "DE")                        |
| `language`            | ISO 639-1-taalcode (bijv. "en", "de")                          |
| `search_lang`         | Zoektaalcode (alleen Brave)                                  |
| `freshness`           | Tijdsfilter: `day`, `week`, `month` of `year`                     |
| `date_after`          | Resultaten na deze datum (YYYY-MM-DD)                               |
| `date_before`         | Resultaten vóór deze datum (YYYY-MM-DD)                              |
| `ui_lang`             | Taalcode van de gebruikersinterface (alleen Brave)                                      |
| `domain_filter`       | Array met toegestane/geblokkeerde domeinen (alleen Perplexity)                  |
| `max_tokens`          | Totaal tokenbudget voor inhoud, alleen de native Perplexity Search API      |
| `max_tokens_per_page` | Tokenlimiet voor extractie per pagina, alleen de native Perplexity Search API |

<Warning>
  Niet alle parameters werken met alle providers. De Brave-modus `llm-context`
  weigert `ui_lang`; `date_before` vereist ook `date_after`, omdat aangepaste
  versheidsbereiken van Brave zowel een begin- als einddatum vereisen.
  Gemini, Grok en Kimi retourneren één samengesteld antwoord met bronvermeldingen. Ze
  accepteren `count` voor compatibiliteit met gedeelde tools, maar dit verandert de
  vorm van het onderbouwde antwoord niet. Gemini behandelt de versheid van `day` als een recentheidssuggestie; ruimere
  versheidswaarden en expliciete datums stellen tijdsbereiken voor Google Search-onderbouwing in.
  Perplexity gedraagt zich op dezelfde manier wanneer je het Sonar/OpenRouter-
  compatibiliteitspad gebruikt (`plugins.entries.perplexity.config.webSearch.baseUrl` /
  `model` of `OPENROUTER_API_KEY`); dat pad biedt ook geen ondersteuning voor `max_tokens` en
  `max_tokens_per_page`.
  SearXNG accepteert `http://` alleen voor vertrouwde hosts in een privénetwerk of op loopback;
  openbare SearXNG-eindpunten moeten `https://` gebruiken.
  Firecrawl en Tavily ondersteunen `query` en `count` alleen via `web_search`
  -- gebruik hun eigen tools voor geavanceerde opties.
</Warning>

## x_search

`x_search` doorzoekt berichten op X (voorheen Twitter) met xAI en retourneert
door AI samengestelde antwoorden met bronvermeldingen. Het accepteert zoekopdrachten in natuurlijke taal en
optionele gestructureerde filters. OpenClaw stelt de ingebouwde xAI-tool `x_search`
per aanvraag samen in plaats van deze permanent geregistreerd te houden, zodat deze alleen
actief is tijdens de beurt waarin de tool daadwerkelijk wordt aangeroepen.

<Warning>
  `x_search` wordt uitgevoerd op de servers van xAI. xAI rekent $5 per 1.000 toolaanroepen, plus de
  invoer- en uitvoertokens van het model.
</Warning>

<Note>
  Volgens de documentatie van xAI ondersteunt `x_search` zoeken op trefwoorden, semantisch zoeken, zoeken naar gebruikers
  en het ophalen van threads. Voor betrokkenheidsstatistieken per bericht, zoals reposts,
  reacties, bladwijzers of weergaven, kun je het beste gericht zoeken naar de exacte URL
  of status-ID van het bericht. Brede zoekopdrachten op trefwoorden kunnen het juiste bericht vinden, maar minder
  volledige metadata per bericht retourneren. Een goed patroon is: zoek eerst het bericht en
  voer daarna een tweede `x_search`-zoekopdracht uit die specifiek op dat bericht is gericht.
</Note>

### Configuratie van x_search

Als `enabled` is weggelaten, wordt `x_search` alleen beschikbaar gesteld wanneer de provider van het actieve model
`xai` is en de xAI-aanmeldgegevens kunnen worden gevonden. Stel voor een actief model met een bekende
niet-xAI-provider `plugins.entries.xai.config.xSearch.enabled` in op `true` om
gebruik tussen providers in te schakelen. Als de provider van het actieve model ontbreekt of
niet kan worden vastgesteld, blijft de tool verborgen. Stel `enabled` in op `false` om de tool voor
elke provider uit te schakelen. xAI-aanmeldgegevens zijn altijd vereist.

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          xSearch: {
            enabled: true, // vereist voor een bekende niet-xAI-modelprovider
            model: "grok-4.3",
            baseUrl: "https://api.x.ai/v1", // optioneel, overschrijft webSearch.baseUrl
            inlineCitations: false,
            maxTurns: 2,
            timeoutSeconds: 30,
            cacheTtlMinutes: 15,
          },
          webSearch: {
            apiKey: "xai-...", // optioneel als een xAI-authenticatieprofiel of XAI_API_KEY is ingesteld
            baseUrl: "https://api.x.ai/v1", // optionele gedeelde basis-URL voor xAI Responses
          },
        },
      },
    },
  },
}
```

`x_search` verstuurt een POST-verzoek naar `<baseUrl>/responses` wanneer
`plugins.entries.xai.config.xSearch.baseUrl` is ingesteld. Als dat veld is weggelaten,
wordt teruggevallen op `plugins.entries.xai.config.webSearch.baseUrl` en vervolgens op het
openbare xAI-eindpunt (`https://api.x.ai/v1`).

### Parameters van x_search

| Parameter                    | Beschrijving                                            |
| ---------------------------- | ------------------------------------------------------ |
| `query`                      | Zoekopdracht (verplicht)                                |
| `allowed_x_handles`          | Beperk resultaten tot maximaal 20 X-handles               |
| `excluded_x_handles`         | Sluit maximaal 20 X-handles uit                           |
| `from_date`                  | Neem alleen berichten op die op of na deze datum zijn geplaatst (YYYY-MM-DD)  |
| `to_date`                    | Neem alleen berichten op die op of vóór deze datum zijn geplaatst (YYYY-MM-DD) |
| `enable_image_understanding` | Laat xAI afbeeldingen inspecteren die aan overeenkomende berichten zijn gekoppeld      |
| `enable_video_understanding` | Laat xAI video's inspecteren die aan overeenkomende berichten zijn gekoppeld      |

`allowed_x_handles` en `excluded_x_handles` sluiten elkaar wederzijds uit.

### Voorbeeld van x_search

```javascript
await x_search({
  query: "dinner recipes",
  allowed_x_handles: ["nytfood"],
  from_date: "2026-03-01",
});
```

```javascript
// Statistieken per bericht: gebruik waar mogelijk de exacte status-URL of status-ID
await x_search({
  query: "https://x.com/huntharo/status/1905678901234567890",
});
```

## Voorbeelden

```javascript
// Eenvoudige zoekopdracht
await web_search({ query: "OpenClaw plugin SDK" });

// Zoekopdracht specifiek voor Duitsland
await web_search({ query: "TV online schauen", country: "DE", language: "de" });

// Recente resultaten (afgelopen week)
await web_search({ query: "AI developments", freshness: "week" });

// Datumbereik
await web_search({
  query: "climate research",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// Domeinfiltering (alleen Perplexity)
await web_search({
  query: "product reviews",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});
```

## Toolprofielen

Als je toolprofielen of toelatingslijsten gebruikt, voeg je `web_search`, `x_search` of `group:web` toe:

```json5
{
  tools: {
    allow: ["web_search", "x_search"],
    // of: allow: ["group:web"]  (omvat web_search, x_search en web_fetch)
  },
}
```

## Gerelateerd

- [Web Fetch](/nl/tools/web-fetch) -- haal een URL op en extraheer leesbare inhoud
- [Webbrowser](/nl/tools/browser) -- volledige browserautomatisering voor sites die veel JavaScript gebruiken
- [Grok Search](/nl/tools/grok-search) -- Grok als de `web_search`-provider
- [Ollama Web Search](/nl/tools/ollama-search) -- zoeken op het web zonder sleutel via je Ollama-host
