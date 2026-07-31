---
read_when:
    - Je wilt Brave Search gebruiken voor web_search
    - Je hebt een BRAVE_API_KEY of abonnementsgegevens nodig
summary: Brave Search API instellen voor web_search
title: Brave-zoekopdracht
x-i18n:
    generated_at: "2026-07-27T06:14:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52168db93abb564eda5868584261e0530ce3cff57c3463a2fc1eded351df30f2
    source_path: tools/brave-search.md
    workflow: 16
---

OpenClaw ondersteunt Brave Search API als `web_search`-provider.

## Een API-sleutel verkrijgen

1. Maak een Brave Search API-account aan op [https://brave.com/search/api/](https://brave.com/search/api/)
2. Kies in het dashboard het **Search**-abonnement en genereer een API-sleutel.
3. Sla de sleutel op in de configuratie of stel `BRAVE_API_KEY` in de Gateway-omgeving in.

## Configuratievoorbeeld

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // of "llm-context"
            baseUrl: "https://api.search.brave.com", // optionele proxy-/basis-URL-overschrijving
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Providerspecifieke Brave-zoekinstellingen staan onder `plugins.entries.brave.config.webSearch.*`; dit is het canonieke configuratiepad.

`webSearch.mode` bepaalt het Brave-transport:

- `web` (standaard): normale Brave-webzoekopdracht met titels, URL's en fragmenten
- `llm-context`: Brave LLM Context API met vooraf geëxtraheerde tekstfragmenten en bronnen voor onderbouwing

`webSearch.baseUrl` kan Brave-verzoeken naar een vertrouwde Brave-compatibele proxy
of gateway sturen. OpenClaw voegt `/res/v1/web/search` of `/res/v1/llm/context` toe aan
de geconfigureerde basis-URL en neemt de basis-URL op in de cachesleutel. Openbare
eindpunten moeten `https://` gebruiken; `http://` wordt alleen geaccepteerd voor vertrouwde loopback-
of proxihosts in een privénetwerk.

## Toolparameters

<ParamField path="query" type="string" required>
Zoekopdracht.
</ParamField>

<ParamField path="count" type="number" default="5">
Aantal te retourneren resultaten (1–10).
</ParamField>

<ParamField path="country" type="string">
ISO-landcode van 2 letters (bijv. `US`, `DE`).
</ParamField>

<ParamField path="language" type="string">
ISO 639-1-taalcode voor zoekresultaten (bijv. `en`, `de`, `fr`).
</ParamField>

<ParamField path="search_lang" type="string">
Brave-code voor de zoektaal (bijv. `en`, `en-gb`, `zh-hans`).
</ParamField>

<ParamField path="ui_lang" type="string">
ISO-taalcode voor UI-elementen.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Tijdfilter — `day` is 24 uur.
</ParamField>

<ParamField path="date_after" type="string">
Alleen resultaten die na deze datum zijn gepubliceerd (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Alleen resultaten die vóór deze datum zijn gepubliceerd (`YYYY-MM-DD`).
</ParamField>

**Voorbeelden:**

```javascript
// Zoekopdracht specifiek voor land en taal
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
```

## Opmerkingen

- OpenClaw gebruikt het Brave **Search**-abonnement. Als je een verouderd abonnement hebt (bijv. het oorspronkelijke Free-abonnement met 2.000 zoekopdrachten/maand), blijft dit geldig, maar bevat het geen nieuwere functies zoals LLM Context of hogere snelheidslimieten.
- Elk Brave-abonnement bevat **\$5/maand aan gratis tegoed** (wordt vernieuwd). Het Search-abonnement kost \$5 per 1.000 verzoeken, dus het tegoed dekt 1.000 zoekopdrachten/maand. Stel je gebruikslimiet in het Brave-dashboard in om onverwachte kosten te voorkomen. Zie de [Brave API-portal](https://brave.com/search/api/) voor de actuele abonnementen.
- Het Search-abonnement bevat het LLM Context-eindpunt en rechten voor AI-inferentie. Voor het opslaan van resultaten om modellen te trainen of af te stemmen, is een abonnement met expliciete opslagrechten vereist. Zie de [Servicevoorwaarden](https://api-dashboard.search.brave.com/terms-of-service) van Brave.
- De modus `llm-context` retourneert onderbouwde bronvermeldingen in plaats van de normale fragmentstructuur voor webzoekopdrachten.
- De modus `llm-context` ondersteunt `freshness` en begrensde bereiken met `date_after` + `date_before`. Deze ondersteunt `ui_lang` niet; `date_before` zonder `date_after` wordt geweigerd, omdat Brave vereist dat aangepaste versheidsbereiken zowel een begin- als einddatum bevatten.
- `ui_lang` moet een regio-subtag bevatten, zoals `en-US`.
- Resultaten worden standaard 15 minuten in de cache opgeslagen (configureerbaar via `cacheTtlMinutes`).
- Aangepaste waarden voor `webSearch.baseUrl` worden opgenomen in de Brave-cache-identiteit, zodat
  proxyspecifieke antwoorden niet conflicteren.
- Schakel de diagnostische vlag `brave.http` in om tijdens probleemoplossing URL's/queryparameters van Brave-verzoeken, de responsstatus/-timing en treffers, missers en schrijfbewerkingen van de zoekcache te loggen. De vlag logt nooit de API-sleutel of responsinhoud, maar zoekopdrachten kunnen gevoelig zijn.

## Gerelateerd

- [Overzicht van Web Search](/nl/tools/web) -- alle providers en automatische detectie
- [Perplexity Search](/nl/tools/perplexity-search) -- gestructureerde resultaten met domeinfiltering
- [Exa Search](/nl/tools/exa-search) -- neuraal zoeken met inhoudsextractie
