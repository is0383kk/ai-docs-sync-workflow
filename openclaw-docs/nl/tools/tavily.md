---
read_when:
    - Je wilt webzoekopdrachten met Tavily als backend
    - Je hebt een Tavily-API-sleutel nodig
    - Je wilt Tavily als web_search-provider gebruiken
    - Je wilt inhoud uit URL's extraheren
summary: Tavily-tools voor zoeken en extraheren
title: Tavily
x-i18n:
    generated_at: "2026-07-27T05:21:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9a61351872eb8aecb0b3ada9b573ee8d3db1dcec3d7bd74074446fbe9dc1f274
    source_path: tools/tavily.md
    workflow: 16
---

[Tavily](https://tavily.com) is een zoek-API die is ontworpen voor AI-toepassingen. OpenClaw biedt deze op twee manieren aan:

- als de `web_search`-provider voor de generieke zoektool
- als expliciete plugintools: `tavily_search` en `tavily_extract`

Tavily retourneert gestructureerde resultaten die zijn geoptimaliseerd voor verwerking door LLM's, met configureerbare zoekdiepte, onderwerpfiltering, domeinfilters, door AI gegenereerde antwoordsamenvattingen en inhoudsextractie uit URL's (inclusief met JavaScript gerenderde pagina's).

| Eigenschap | Waarde                                                                                        |
| ---------- | --------------------------------------------------------------------------------------------- |
| Plugin-id  | `tavily`                                                                            |
| Pakket     | `@openclaw/tavily-plugin`                                                                            |
| Authenticatie | Omgevingsvariabele `TAVILY_API_KEY` of configuratie `apiKey`                   |
| Basis-URL  | `https://api.tavily.com` (standaard); omgevingsvariabele `TAVILY_BASE_URL` of configuratie `baseUrl` om deze te overschrijven |
| Time-outs  | 30s voor zoeken, 60s voor extractie (standaard)                                                |
| Tools      | `tavily_search`, `tavily_extract`                                                        |

## Aan de slag

<Steps>
  <Step title="Installeer de plugin">
    ```bash
    openclaw plugins install @openclaw/tavily-plugin
    ```
  </Step>
  <Step title="Verkrijg een API-sleutel">
    Maak een Tavily-account aan op [tavily.com](https://tavily.com) en genereer vervolgens een API-sleutel in het dashboard.
  </Step>
  <Step title="Configureer de plugin en provider">
    ```json5
    {
      plugins: {
        entries: {
          tavily: {
            enabled: true,
            config: {
              webSearch: {
                apiKey: "tvly-...", // optioneel als TAVILY_API_KEY is ingesteld
                baseUrl: "https://api.tavily.com",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            provider: "tavily",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Controleer of zoekopdrachten worden uitgevoerd">
    Activeer een `web_search` vanuit een willekeurige agent of roep `tavily_search` rechtstreeks aan.
  </Step>
</Steps>

<Tip>
Als je Tavily kiest tijdens de onboarding of in `openclaw configure --section web`, wordt de officiële Tavily-plugin indien nodig geïnstalleerd en ingeschakeld.
</Tip>

## Toolreferentie

### `tavily_search`

Gebruik dit wanneer je Tavily-specifieke zoekinstellingen wilt in plaats van de generieke `web_search`.

| Parameter         | Type         | Beperkingen / standaard                | Beschrijving                                  |
| ----------------- | ------------ | -------------------------------------- | --------------------------------------------- |
| `query`           | tekenreeks   | vereist                                | Tekenreeks met de zoekopdracht.               |
| `search_depth`    | opsomming    | `basic` (standaard), `advanced`          | `advanced` is langzamer, maar biedt een hogere relevantie. |
| `topic`           | opsomming    | `general` (standaard), `news`, `finance` | Filter op onderwerpcategorie.                 |
| `max_results`     | geheel getal | 1-20, standaard `5`     | Aantal resultaten.                            |
| `include_answer`  | booleaans     | standaard `false`           | Neem een door Tavily AI gegenereerde antwoordsamenvatting op. |
| `time_range`      | opsomming    | `day`, `week`, `month`, `year` | Filter resultaten op recentheid.             |
| `include_domains` | tekenreeksarray | (geen)                               | Neem alleen resultaten uit deze domeinen op.  |
| `exclude_domains` | tekenreeksarray | (geen)                               | Sluit resultaten uit deze domeinen uit.       |

Afweging voor zoekdiepte:

| Diepte     | Snelheid | Relevantie | Meest geschikt voor                  |
| ---------- | -------- | ---------- | ------------------------------------ |
| `basic`    | Sneller  | Hoog       | Algemene zoekopdrachten (standaard). |
| `advanced` | Langzamer | Hoogst     | Nauwkeurig onderzoek en feitenonderzoek. |

### `tavily_extract`

Gebruik dit om opgeschoonde inhoud uit een of meer URL's te extraheren. Verwerkt met JavaScript gerenderde pagina's en ondersteunt op zoekopdrachten gerichte segmentering voor doelgerichte extractie.

| Parameter           | Type           | Beperkingen / standaard       | Beschrijving                                                |
| ------------------- | -------------- | ----------------------------- | ----------------------------------------------------------- |
| `urls`              | tekenreeksarray | vereist, 1-20              | URL's waaruit inhoud moet worden geëxtraheerd.               |
| `query`             | tekenreeks     | (optioneel)                  | Rangschik geëxtraheerde segmenten opnieuw op relevantie voor deze zoekopdracht. |
| `extract_depth`     | opsomming      | `basic` (standaard), `advanced` | Gebruik `advanced` voor pagina's met veel JS, SPA's of dynamische tabellen. |
| `chunks_per_source` | geheel getal   | 1-5; **vereist `query`** | Aantal geretourneerde segmenten per URL. Geeft een fout als dit zonder `query` wordt ingesteld. |
| `include_images`    | booleaans       | standaard `false` | Neem afbeeldings-URL's op in de resultaten.                 |

Afweging voor extractiediepte:

| Diepte     | Wanneer te gebruiken                       |
| ---------- | ------------------------------------------ |
| `basic`    | Eenvoudige pagina's. Probeer dit eerst.    |
| `advanced` | Met JS gerenderde SPA's, dynamische inhoud en tabellen. |

<Tip>
Verdeel grotere URL-lijsten over meerdere aanroepen van `tavily_extract` (maximaal 20 per aanvraag). Gebruik `query` samen met `chunks_per_source` om alleen relevante inhoud te verkrijgen in plaats van volledige pagina's.
</Tip>

## De juiste tool kiezen

| Behoefte                              | Tool                         |
| ------------------------------------- | ---------------------------- |
| Snel zoeken op het web, zonder speciale opties | `web_search`     |
| Zoeken met diepte, onderwerp en AI-antwoorden | `tavily_search`     |
| Inhoud uit specifieke URL's extraheren | `tavily_extract`           |

<Note>
De generieke tool `web_search` met Tavily als provider ondersteunt `query` en `count` (maximaal 20 resultaten). Gebruik voor Tavily-specifieke instellingen (`search_depth`, `topic`, `include_answer`, domeinfilters en tijdsbereik) in plaats daarvan `tavily_search`.
</Note>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Volgorde voor het bepalen van de API-sleutel">
    De Tavily-client zoekt de API-sleutel in deze volgorde op:

    1. `plugins.entries.tavily.config.webSearch.apiKey` (opgelost via SecretRefs).
    2. `TAVILY_API_KEY` uit de Gateway-omgeving.

    `tavily_search` en `tavily_extract` geven beide een configuratiefout als geen van beide aanwezig is.

  </Accordion>

  <Accordion title="Aangepaste basis-URL">
    Overschrijf `plugins.entries.tavily.config.webSearch.baseUrl` of stel `TAVILY_BASE_URL` in als je Tavily via een proxy aanbiedt. De configuratie heeft voorrang op de omgevingsvariabele. De standaardwaarde is `https://api.tavily.com`.
  </Accordion>

  <Accordion title="`chunks_per_source` vereist `query`">
    `tavily_extract` weigert aanroepen die `chunks_per_source` doorgeven zonder een `query`. Tavily rangschikt segmenten op relevantie voor de zoekopdracht, waardoor de parameter zonder zoekopdracht betekenisloos is.
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Overzicht van zoeken op het web" href="/nl/tools/web" icon="magnifying-glass">
    Alle providers en regels voor automatische detectie.
  </Card>
  <Card title="Firecrawl" href="/nl/tools/firecrawl" icon="fire">
    Zoeken en scrapen met inhoudsextractie.
  </Card>
  <Card title="Exa Search" href="/nl/tools/exa-search" icon="binoculars">
    Neuraal zoeken met inhoudsextractie.
  </Card>
  <Card title="Configuratie" href="/nl/gateway/configuration" icon="gear">
    Volledig configuratieschema voor pluginvermeldingen en toolroutering.
  </Card>
</CardGroup>
