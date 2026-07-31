---
read_when:
    - Je wilt een zelfgehoste webzoekprovider
    - Je wilt SearXNG gebruiken voor web_search
    - Je hebt een privacygerichte of air-gapped zoekoptie nodig
summary: SearXNG-webzoekfunctie -- zelfgehoste metazoekprovider zonder API-sleutel
title: SearXNG-zoekopdracht
x-i18n:
    generated_at: "2026-07-27T06:16:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cae8de9f8e2c8dd9cec615adb48da5c1fd7654bffe96c7afc1acea3effbcf1fc
    source_path: tools/searxng-search.md
    workflow: 16
---

OpenClaw ondersteunt [SearXNG](https://docs.searxng.org/) als een **zelfgehoste
provider zonder sleutel** voor `web_search`. SearXNG is een opensource-metazoekmachine
die resultaten van Google, Bing, DuckDuckGo en andere bronnen samenvoegt.

Voordelen:

- **Gratis en onbeperkt** -- geen API-sleutel of commercieel abonnement vereist
- **Privacy / airgap** -- zoekopdrachten verlaten je netwerk nooit
- **Werkt overal** -- geen regiobeperkingen van commerciële zoek-API's

## Installatie

<Steps>
  <Step title="Installeer de Plugin">
    ```bash
    openclaw plugins install @openclaw/searxng-plugin
    ```
  </Step>
  <Step title="Voer een SearXNG-instantie uit">
    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    Je kunt ook een bestaande SearXNG-implementatie gebruiken waartoe je toegang hebt. Raadpleeg de
    [SearXNG-documentatie](https://docs.searxng.org/) voor de productieconfiguratie.

  </Step>
  <Step title="Configureer">
    ```bash
    openclaw configure --section web
    # Selecteer "searxng" als provider
    ```

    Je kunt ook de omgevingsvariabele instellen en deze via automatische detectie laten vinden:

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

  </Step>
</Steps>

## Configuratie

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

Instellingen op Plugin-niveau voor de SearXNG-instantie:

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // optioneel
            language: "en", // optioneel
          },
        },
      },
    },
  },
}
```

`baseUrl` accepteert ook een SecretRef-object (bijvoorbeeld `{ source: "env", id: "SEARXNG_BASE_URL" }`).

## Omgevingsvariabele

Stel `SEARXNG_BASE_URL` in als alternatief voor de configuratie:

```bash
export SEARXNG_BASE_URL="http://localhost:8888"
```

Volgorde van verwerking: de geconfigureerde tekenreeks `baseUrl`, vervolgens een inline SecretRef voor een omgevingsvariabele op
`baseUrl` en daarna `SEARXNG_BASE_URL`. Wanneer geen van de configuratiepaden is ingesteld,
`SEARXNG_BASE_URL` aanwezig is en er geen expliciete provider is gekozen, selecteert automatische detectie
SearXNG.

## Referentie voor Plugin-configuratie

| Veld        | Beschrijving                                                        |
| ------------ | ------------------------------------------------------------------ |
| `baseUrl`    | Basis-URL van je SearXNG-instantie (vereist)                       |
| `categories` | Door komma's gescheiden categorieën, zoals `general`, `news` of `science` |
| `language`   | Taalcode voor resultaten, zoals `en`, `de` of `fr`              |

De toolaanroep `web_search` accepteert ook `count` (1-10 resultaten), `categories`
en `language` als overschrijvingen per aanroep.

## Opmerkingen

- **JSON-API** -- gebruikt het native `format=json`-eindpunt van SearXNG, niet het scrapen van HTML
- **URL's van afbeeldingsresultaten** -- resultaten uit de afbeeldingscategorie bevatten `img_src` wanneer SearXNG
  een rechtstreekse afbeeldings-URL retourneert
- **Geen API-sleutel** -- werkt direct met elke SearXNG-instantie
- **Validatie van basis-URL** -- `baseUrl` moet een geldige `http://`- of `https://`-
  URL zijn
- **Netwerkbeveiliging** -- `http://`-basis-URL's moeten naar een vertrouwde privé- of
  loopbackhost verwijzen (openbare hosts moeten `https://` gebruiken); `https://`-basis-URL's die
  naar een privé-/intern adres worden omgezet, krijgen dezelfde vrijstelling voor zelfgehoste instanties,
  terwijl `https://`-basis-URL's die naar een openbaar adres worden omgezet, strikte SSRF-bescherming behouden
- **Volgorde van automatische detectie** -- SearXNG vereist een geconfigureerde `baseUrl` (volgnummer
  200 onder providers die al over hun vereiste referentie beschikken). Providers zonder sleutel,
  zoals DuckDuckGo of Ollama Web Search, worden nooit impliciet door automatische detectie gekozen;
  ze worden alleen geactiveerd bij een expliciete keuze voor `provider`
- **Zelfgehost** -- je beheert de instantie, zoekopdrachten en bovenliggende zoekmachines
- **Categorieën** gebruiken standaard `general` wanneer ze niet zijn geconfigureerd
- **Terugvalcategorie** -- als een categorieaanvraag die niet voor `general` is, slaagt maar
  nul resultaten oplevert, probeert OpenClaw dezelfde zoekopdracht één keer opnieuw met `general`
  voordat een lege resultatenset wordt geretourneerd
- **Resultaatcaching** -- identieke zoekopdrachten (dezelfde zoekopdracht, hetzelfde aantal, dezelfde categorieën,
  taal en basis-URL) worden gedurende een korte TTL binnen het proces gecachet
- **Versievereiste** -- de Plugin declareert `minHostVersion: >=2026.6.9`

<Tip>
  Zorg ervoor dat de `json`-indeling in `settings.yml` onder `search.formats`
  is ingeschakeld in je SearXNG-instantie, zodat de JSON-API van SearXNG werkt.
</Tip>

## Gerelateerd

- [Overzicht van zoeken op het web](/nl/tools/web) -- alle providers en automatische detectie
- [Zoeken met DuckDuckGo](/nl/tools/duckduckgo-search) -- nog een provider zonder sleutel
- [Zoeken met Brave](/nl/tools/brave-search) -- gestructureerde resultaten met een gratis abonnementscategorie
