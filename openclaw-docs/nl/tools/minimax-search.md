---
read_when:
    - Je wilt MiniMax gebruiken voor web_search
    - Je hebt een MiniMax Token Plan-sleutel of OAuth-token nodig
    - Je wilt richtlijnen voor de MiniMax CN/global-zoekhost
summary: MiniMax Search via de zoek-API van Token Plan
title: MiniMax-zoekfunctie
x-i18n:
    generated_at: "2026-07-27T05:20:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb851614bbe43f011e07fe3e80d5390f1ba515f3e00ba749c91999617ad2d1e2
    source_path: tools/minimax-search.md
    workflow: 16
---

OpenClaw ondersteunt MiniMax als een `web_search`-provider via de zoek-API van het MiniMax
Token Plan. Deze retourneert gestructureerde zoekresultaten met titels, URL's,
fragmenten en gerelateerde zoekopdrachten.

## Een Token Plan-referentie verkrijgen

<Steps>
  <Step title="Een sleutel maken">
    Maak of kopieer een MiniMax Token Plan-sleutel via
    [MiniMax Platform](https://platform.minimax.io/user-center/basic-information/interface-key).
    OAuth-configuraties kunnen in plaats daarvan `MINIMAX_OAUTH_TOKEN` hergebruiken.
  </Step>
  <Step title="De sleutel opslaan">
    Stel `MINIMAX_CODE_PLAN_KEY` in de Gateway-omgeving in of configureer deze via:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

OpenClaw accepteert ook `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN` en
`MINIMAX_API_KEY` als omgevingsvariabele-aliassen. Deze worden in die volgorde gecontroleerd na
`MINIMAX_CODE_PLAN_KEY`. `MINIMAX_API_KEY` moet verwijzen naar een
Token Plan-referentie waarvoor zoeken is ingeschakeld; gewone API-sleutels voor MiniMax-modellen worden mogelijk niet geaccepteerd door
het Token Plan-zoekeindpunt.

## Configuratie

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // optioneel als een MiniMax Token Plan-omgevingsvariabele is ingesteld
            region: "global", // of "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

**Alternatief via de omgeving:** stel `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`,
`MINIMAX_OAUTH_TOKEN` of `MINIMAX_API_KEY` in de Gateway-omgeving in.
Plaats deze bij een Gateway-installatie in `~/.openclaw/.env`.

## Regio selecteren

MiniMax Search gebruikt deze eindpunten:

- Wereldwijd: `https://api.minimax.io/v1/coding_plan/search`
- CN: `https://api.minimaxi.com/v1/coding_plan/search`

Als `plugins.entries.minimax.config.webSearch.region` niet is ingesteld, bepaalt OpenClaw
de regio in deze volgorde:

1. `webSearch.region` van de Plugin
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

Dit betekent dat onboarding voor CN of `MINIMAX_API_HOST=https://api.minimaxi.com/...`
MiniMax Search automatisch ook op de CN-host houdt.

Zelfs wanneer je MiniMax hebt geverifieerd via het OAuth-pad `minimax-portal`,
wordt zoeken op het web nog steeds geregistreerd met provider-id `minimax`; de basis-URL van de OAuth-provider
wordt gebruikt als regiohint voor de selectie van de CN- of wereldwijde host, en `MINIMAX_OAUTH_TOKEN`
kan dienen als bearer-referentie voor MiniMax Search.

## Ondersteunde parameters

| Parameter | Type    | Beperkingen     | Beschrijving                                                                 |
| --------- | ------- | --------------- | --------------------------------------------------------------------------- |
| `query`   | tekenreeks  | vereist        | Tekenreeks met de zoekopdracht.                                                        |
| `count`   | geheel getal | 1-10, standaard 5 | Aantal te retourneren resultaten. OpenClaw kort de geretourneerde lijst in tot deze grootte. |

Providerspecifieke filters worden momenteel niet ondersteund.

## Gerelateerd

- [Overzicht van zoeken op het web](/nl/tools/web) -- alle providers en automatische detectie
- [MiniMax](/nl/providers/minimax) -- configuratie van modellen, afbeeldingen, spraak en verificatie
