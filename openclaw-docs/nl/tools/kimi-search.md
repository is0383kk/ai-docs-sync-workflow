---
read_when:
    - Je wilt Kimi gebruiken voor web_search
    - Je hebt een KIMI_API_KEY of MOONSHOT_API_KEY nodig
summary: Kimi-webzoekopdracht via Moonshot-webzoekopdracht
title: Kimi-zoekopdracht
x-i18n:
    generated_at: "2026-07-27T05:36:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 65e5f8c9f3b607dbcc3256c51a6a083864e31f65ed2a751d2d500abeb35ba844
    source_path: tools/kimi-search.md
    workflow: 16
---

Kimi is een `web_search`-provider die wordt ondersteund door de native webzoekfunctie van Moonshot. Moonshot
stelt één antwoord met inline bronvermeldingen samen, vergelijkbaar met de providers voor
onderbouwde antwoorden van Gemini en Grok, in plaats van een gerangschikte resultatenlijst te retourneren.

## Instellen

<Steps>
  <Step title="Een sleutel maken">
    Haal een API-sleutel op bij [Moonshot AI](https://platform.moonshot.cn/).
  </Step>
  <Step title="De sleutel opslaan">
    Stel `KIMI_API_KEY` of `MOONSHOT_API_KEY` in de Gateway-omgeving in (voeg deze voor een
    Gateway-installatie toe aan `~/.openclaw/.env`), of configureer dit via:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

Wanneer je **Kimi** kiest tijdens `openclaw onboard` of `openclaw configure --section web`,
wordt ook gevraagd om:

- de Moonshot API-regio: `https://api.moonshot.ai/v1` of `https://api.moonshot.cn/v1`
- het webzoekmodel (standaard `kimi-k2.6`)

## Configuratie

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // optioneel als KIMI_API_KEY of MOONSHOT_API_KEY is ingesteld
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

`tools.web.search.provider` wordt bij weglating automatisch gedetecteerd aan de hand van beschikbare API-sleutels;
stel dit expliciet in op `kimi` als meerdere zoekreferenties zijn geconfigureerd.

Configureer Kimi-specifieke waarden voor `apiKey`, `baseUrl` en `model` onder
`plugins.entries.moonshot.config.webSearch`.

Standaardwaarden: `baseUrl` is bij weglating standaard `https://api.moonshot.ai/v1`, `model`
is standaard `kimi-k2.6`.

Als chatverkeer de Chinese host gebruikt (`models.providers.moonshot.baseUrl`:
`https://api.moonshot.cn/v1`), hergebruikt Kimi `web_search` die host automatisch
wanneer de eigen `baseUrl` niet is ingesteld, zodat `.cn`-sleutels niet per ongeluk het
internationale eindpunt aanroepen (dat HTTP 401 retourneert voor die sleutels). Stel een expliciete
Kimi-waarde voor `baseUrl` in om deze overname te negeren.

## Vereiste voor onderbouwing

OpenClaw retourneert een Kimi-resultaat voor `web_search` pas nadat het antwoord van Moonshot
native onderbouwing uit de webzoekfunctie bevat, zoals een herhaling van een `$web_search`-toolaanroep,
`search_results` of URL's van bronvermeldingen. Als Kimi rechtstreeks antwoordt zonder
onderbouwing (bijvoorbeeld "Ik kan niet op internet browsen"), retourneert OpenClaw een
`kimi_web_search_ungrounded`-fout in plaats van die tekst als zoekresultaat te
behandelen. Probeer de zoekopdracht opnieuw, schakel over naar een gestructureerde provider zoals Brave, of gebruik
`web_fetch` / de browsertool wanneer je al een doel-URL hebt.

## Toolparameters

| Parameter                                                       | Ondersteund                                                                                                                        |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `query`                                                         | Ja                                                                                                                                 |
| `count`                                                         | Geaccepteerd voor compatibiliteit tussen providers, maar genegeerd: Kimi retourneert altijd één samengesteld antwoord, geen lijst met N resultaten |
| `country`, `language`, `freshness`, `date_after`, `date_before` | Nee                                                                                                                                |

## Gerelateerd

- [Overzicht van zoeken op het web](/nl/tools/web) - alle providers en automatische detectie
- [Moonshot AI](/nl/providers/moonshot) - documentatie voor het Moonshot-model en de Kimi Coding-provider
- [Gemini Search](/nl/tools/gemini-search) - door AI samengestelde antwoorden via onderbouwing van Google
- [Grok Search](/nl/tools/grok-search) - door AI samengestelde antwoorden via onderbouwing van xAI
