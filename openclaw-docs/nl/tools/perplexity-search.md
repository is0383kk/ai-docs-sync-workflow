---
read_when:
    - Je wilt Perplexity Search gebruiken voor zoeken op het web
    - Je moet PERPLEXITY_API_KEY of OPENROUTER_API_KEY instellen
summary: Perplexity Search API en Sonar/OpenRouter-compatibiliteit voor web_search
title: Perplexity-zoekopdracht
x-i18n:
    generated_at: "2026-07-27T05:20:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7ca97355110e70a05f1d57acab475dda8dec89393804df40c6e9be5e30780e8
    source_path: tools/perplexity-search.md
    workflow: 16
---

OpenClaw ondersteunt de Perplexity Search API als een `web_search`-provider. Deze retourneert gestructureerde resultaten met de velden `title`, `url` en `snippet`.

Voor compatibiliteit ondersteunt OpenClaw ook verouderde Perplexity Sonar/OpenRouter-configuraties. Als je `OPENROUTER_API_KEY` gebruikt, een `sk-or-...`-sleutel in `plugins.entries.perplexity.config.webSearch.apiKey` hebt of `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` instelt, schakelt de provider over naar het pad voor chat-completions en retourneert deze door AI samengestelde antwoorden met bronverwijzingen in plaats van gestructureerde resultaten van de Search API.

## Plugin installeren

Installeer de officiële Plugin en start daarna de Gateway opnieuw:

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## Een Perplexity API-sleutel verkrijgen

1. Maak een Perplexity-account aan op [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api).
2. Genereer een API-sleutel in het dashboard.
3. Sla de sleutel op in de configuratie of stel `PERPLEXITY_API_KEY` in de Gateway-omgeving in.

## OpenRouter-compatibiliteit

Als je OpenRouter al voor Perplexity Sonar gebruikte, behoud dan `provider: "perplexity"` en stel `OPENROUTER_API_KEY` in de Gateway-omgeving in, of sla een `sk-or-...`-sleutel op in `plugins.entries.perplexity.config.webSearch.apiKey`.

Optionele compatibiliteitsinstellingen:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## Configuratievoorbeelden

### Native Perplexity Search API

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### OpenRouter-/Sonar-compatibiliteit

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## Waar je de sleutel instelt

**Via de configuratie:** voer `openclaw configure --section web` uit. Hiermee wordt de sleutel opgeslagen in `~/.openclaw/openclaw.json` onder `plugins.entries.perplexity.config.webSearch.apiKey`. Dit veld accepteert ook SecretRef-objecten.

**Via de omgeving:** stel `PERPLEXITY_API_KEY` of `OPENROUTER_API_KEY` in de procesomgeving van de Gateway in. Plaats deze voor een Gateway-installatie in `~/.openclaw/.env` (of in je serviceomgeving). Zie [Omgevingsvariabelen](/nl/help/faq#env-vars-and-env-loading).

Als `provider: "perplexity"` is geconfigureerd en de SecretRef van de Perplexity-sleutel niet kan worden omgezet en er geen terugvaloptie via een omgevingsvariabele is, mislukt het opstarten of opnieuw laden onmiddellijk.

## Toolparameters

Deze parameters zijn van toepassing op het native pad van de Perplexity Search API.

<ParamField path="query" type="string" required>
Zoekopdracht.
</ParamField>

<ParamField path="count" type="number" default="5">
Aantal te retourneren resultaten (1-10).
</ParamField>

<ParamField path="country" type="string">
ISO-landcode van 2 letters (bijv. `US`, `DE`).
</ParamField>

<ParamField path="language" type="string">
ISO 639-1-taalcode (bijv. `en`, `de`, `fr`).
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Tijdfilter - `day` is 24 uur.
</ParamField>

<ParamField path="date_after" type="string">
Alleen resultaten die na deze datum zijn gepubliceerd (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Alleen resultaten die vóór deze datum zijn gepubliceerd (`YYYY-MM-DD`).
</ParamField>

<ParamField path="domain_filter" type="string[]">
Array met toegestane/geblokkeerde domeinen (maximaal 20).
</ParamField>

<ParamField path="max_tokens" type="number" default="25000">
Totaal inhoudsbudget (maximaal 1000000).
</ParamField>

<ParamField path="max_tokens_per_page" type="number" default="2048">
Tokenlimiet per pagina.
</ParamField>

Voor het verouderde compatibiliteitspad voor Sonar/OpenRouter:

- `query`, `count` en `freshness` worden geaccepteerd.
- `count` dient daar alleen voor compatibiliteit; het antwoord blijft één samengesteld antwoord met bronverwijzingen in plaats van een lijst met N resultaten.
- Filters die uitsluitend voor de Search API bestemd zijn (`country`, `language`, `date_after`, `date_before`, `domain_filter`, `max_tokens`, `max_tokens_per_page`) retourneren expliciete fouten.

**Voorbeelden:**

```javascript
// Land- en taalspecifieke zoekopdracht
await web_search({
  query: "hernieuwbare energie",
  country: "DE",
  language: "de",
});

// Recente resultaten (afgelopen week)
await web_search({
  query: "AI-nieuws",
  freshness: "week",
});

// Zoeken binnen een datumbereik
await web_search({
  query: "AI-ontwikkelingen",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// Domeinen filteren (toegestane lijst)
await web_search({
  query: "klimaatonderzoek",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// Domeinen filteren (blokkeerlijst - gebruik - als voorvoegsel)
await web_search({
  query: "productbeoordelingen",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// Meer inhoud extraheren
await web_search({
  query: "gedetailleerd AI-onderzoek",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### Regels voor domeinfilters

- Maximaal 20 domeinen per filter.
- Je kunt vermeldingen uit de lijst met toegestane domeinen en de blokkeerlijst niet in hetzelfde verzoek combineren.
- Gebruik het voorvoegsel `-` voor vermeldingen in de blokkeerlijst (bijv. `["-reddit.com"]`).

## Opmerkingen

- De Perplexity Search API retourneert gestructureerde webzoekresultaten (`title`, `url`, `snippet`).
- OpenRouter, of een expliciete `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`, schakelt Perplexity voor compatibiliteit terug naar Sonar-chat-completions.
- Sonar-/OpenRouter-compatibiliteit retourneert één samengesteld antwoord met bronverwijzingen, geen gestructureerde resultaatrijen.
- Resultaten worden standaard 15 minuten in de cache opgeslagen (configureerbaar via `cacheTtlMinutes`).

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Overzicht van zoeken op het web" href="/nl/tools/web" icon="globe">
    Alle providers en regels voor automatische detectie.
  </Card>
  <Card title="Zoeken met Brave" href="/nl/tools/brave-search" icon="shield">
    Gestructureerde resultaten met land- en taalfilters.
  </Card>
  <Card title="Zoeken met Exa" href="/nl/tools/exa-search" icon="magnifying-glass">
    Neuraal zoeken met inhoudsextractie.
  </Card>
  <Card title="Documentatie van de Perplexity Search API" href="https://docs.perplexity.ai/docs/search/quickstart" icon="arrow-up-right-from-square">
    Officiële snelstart en naslaginformatie voor de Perplexity Search API.
  </Card>
</CardGroup>
