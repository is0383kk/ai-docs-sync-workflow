---
read_when:
    - Je wilt een webzoekprovider waarvoor geen API-sleutel nodig is
    - Je wilt DuckDuckGo gebruiken voor web_search
    - Je wilt een expliciet geselecteerde zoekprovider zonder sleutel
summary: DuckDuckGo-webzoekfunctie -- provider zonder sleutel (experimenteel, op HTML gebaseerd)
title: DuckDuckGo-zoekopdracht
x-i18n:
    generated_at: "2026-07-27T06:15:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84e90532de276dcb3f73c67015dffe5f5a62be673e44a19053b2b1dfcb0986ac
    source_path: tools/duckduckgo-search.md
    workflow: 16
---

OpenClaw ondersteunt DuckDuckGo als een **sleutelvrije** `web_search`-provider. Er is geen API-sleutel of account vereist.

<Warning>
  DuckDuckGo is een **experimentele, niet-officiële** integratie die de HTML-zoekpagina's zonder JavaScript van DuckDuckGo scrapt -- het is geen officiële API. Houd rekening met incidentele uitval door pagina's met botcontroles of HTML-wijzigingen.
</Warning>

## Instellen

DuckDuckGo wordt nooit automatisch geselecteerd, omdat automatische detectie alleen rekening houdt met providers met bruikbare referenties. Stel de provider expliciet in:

<Steps>
  <Step title="Configureren">
    ```bash
    openclaw configure --section web
    # Selecteer "duckduckgo" als provider
    ```
  </Step>
</Steps>

## Configuratie

Stel de provider rechtstreeks in de configuratie in:

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

Optionele instellingen op Plugin-niveau voor regio en SafeSearch:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // DuckDuckGo-regiocode
            safeSearch: "moderate", // "strict", "moderate" of "off"
          },
        },
      },
    },
  },
}
```

## Toolparameters

<ParamField path="query" type="string" required>
Zoekopdracht.
</ParamField>

<ParamField path="count" type="number" default="5">
Aantal te retourneren resultaten (1-10).
</ParamField>

<ParamField path="region" type="string">
DuckDuckGo-regiocode (bijv. `us-en`, `uk-en`, `de-de`).
</ParamField>

<ParamField path="safeSearch" type="'strict' | 'moderate' | 'off'" default="moderate">
SafeSearch-niveau.
</ParamField>

De toolparameters `region` en `safeSearch` overschrijven per zoekopdracht de bovenstaande configuratiewaarden van de Plugin.

## Opmerkingen

- **Geen API-sleutel** -- werkt zodra DuckDuckGo als `web_search`-provider is geselecteerd.
- **Experimenteel** -- scrapt de HTML-zoekpagina's zonder JavaScript van DuckDuckGo en gebruikt geen officiële API of SDK. Resultaten zijn afhankelijk van de paginastructuur, die zonder voorafgaande kennisgeving kan veranderen.
- **Risico op botcontroles** -- DuckDuckGo kan CAPTCHA's tonen of verzoeken blokkeren bij intensief of geautomatiseerd gebruik.
- **Alleen expliciete selectie** -- de automatische detectie van OpenClaw houdt alleen rekening met providers met bruikbare referenties. Daarom wordt een sleutelvrije provider zoals DuckDuckGo nooit automatisch gekozen; je moet `provider: "duckduckgo"` instellen.
- **SafeSearch gebruikt standaard `moderate`** wanneer dit niet is geconfigureerd.

<Tip>
  Overweeg voor productiegebruik [Brave Search](/nl/tools/brave-search) (gratis niveau beschikbaar) of een andere provider op basis van een API.
</Tip>

## Gerelateerd

- [Overzicht van zoeken op het web](/nl/tools/web) -- alle providers en automatische detectie
- [Brave Search](/nl/tools/brave-search) -- gestructureerde resultaten met een gratis niveau
- [Exa Search](/nl/tools/exa-search) -- neuraal zoeken met inhoudsextractie
