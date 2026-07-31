---
read_when:
    - Je wilt Grok gebruiken voor web_search
    - Je wilt xAI OAuth of een XAI_API_KEY gebruiken voor zoeken op internet
summary: Grok-webzoekopdrachten via webgebaseerde antwoorden van xAI
title: Grok-zoekopdracht
x-i18n:
    generated_at: "2026-07-27T06:36:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e39edd660d0ffe8be066ae81317810da691a7dbd8c59a74222a59145cff5c77
    source_path: tools/grok-search.md
    workflow: 16
---

OpenClaw ondersteunt Grok als een `web_search`-provider en gebruikt daarbij op het web gebaseerde
antwoorden van xAI om door AI samengestelde antwoorden te produceren die worden onderbouwd door actuele zoekresultaten
met bronvermeldingen.

Grok-webzoekopdrachten geven de voorkeur aan een bestaande xAI OAuth-aanmelding wanneer die beschikbaar is.
Als er geen OAuth-profiel bestaat, wordt dezelfde xAI API-sleutel ook gebruikt voor de ingebouwde
`x_search`-tool voor het zoeken naar berichten op X (voorheen Twitter) en de `code_execution`-
tool. Door de sleutel op te slaan in `plugins.entries.xai.config.webSearch.apiKey` kan
OpenClaw deze ook hergebruiken als fallback voor de meegeleverde xAI-modelprovider.

Gebruik voor statistieken op berichtniveau van X (reposts, antwoorden, bladwijzers, weergaven)
[`x_search`](/nl/tools/web#x_search) met de exacte bericht-URL of status-ID
in plaats van een brede zoekopdracht.

## Onboarding en configuratie

Als je **Grok** kiest tijdens `openclaw onboard` of `openclaw configure --section
web`, kan OpenClaw een bestaand xAI OAuth-profiel hergebruiken zonder om
een afzonderlijke sleutel voor webzoekopdrachten te vragen. Zonder OAuth valt het terug op de configuratie van een xAI API-sleutel.

OpenClaw biedt vervolgens een vervolgstap om `x_search` in te schakelen met dezelfde xAI-
referentie. Deze vervolgstap:

- verschijnt alleen nadat je Grok kiest voor `web_search`
- is geen afzonderlijke providerkeuze voor webzoekopdrachten op het hoogste niveau
- kan optioneel het `x_search`-model instellen binnen dezelfde flow

Sla deze stap over om `x_search` later in de configuratie in te schakelen of te wijzigen.

## Aanmelden of een API-sleutel verkrijgen

<Steps>
  <Step title="xAI OAuth gebruiken">
    Als je je tijdens de onboarding of modelauthenticatie al bij xAI hebt aangemeld, kies je
    Grok als de `web_search`-provider. Er is geen afzonderlijke API-sleutel vereist:

    ```bash
    openclaw onboard --auth-choice xai-oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Step>
  <Step title="Een API-sleutel als fallback gebruiken">
    Verkrijg een API-sleutel van [xAI](https://console.x.ai/) wanneer OAuth niet beschikbaar is
    of wanneer je bewust een door een sleutel ondersteunde configuratie voor webzoekopdrachten wilt.
  </Step>
  <Step title="De sleutel opslaan">
    Stel `XAI_API_KEY` in de Gateway-omgeving in of configureer deze via:

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
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // optioneel als xAI OAuth of XAI_API_KEY beschikbaar is
            baseUrl: "https://api.x.ai/v1", // optionele overschrijving van de proxy-/basis-URL voor de Responses API
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**Alternatieve referenties:** `openclaw models auth login --provider xai
--method oauth`, `XAI_API_KEY` in de Gateway-omgeving of
`plugins.entries.xai.config.webSearch.apiKey`. Plaats voor een Gateway-installatie de omgevingsvariabelen
in `~/.openclaw/.env`.

## Hoe het werkt

Grok gebruikt op het web gebaseerde antwoorden van xAI om antwoorden met inline
bronvermeldingen samen te stellen, vergelijkbaar met Gemini's aanpak voor onderbouwing via Google Zoeken.

## Ondersteunde parameters

Grok-zoekopdrachten ondersteunen `query`. `count` wordt geaccepteerd voor gedeelde compatibiliteit met `web_search`,
maar Grok retourneert altijd één samengesteld antwoord met bronvermeldingen
in plaats van een lijst met N resultaten. Providerspecifieke filters worden niet ondersteund.

Grok gebruikt standaard een time-out van 60 seconden, omdat op het web gebaseerde
zoekopdrachten via xAI Responses langer kunnen duren dan de gedeelde standaardwaarde van `web_search`. Overschrijf deze
met `tools.web.search.timeoutSeconds`.

## Overschrijvingen van de basis-URL

Stel `plugins.entries.xai.config.webSearch.baseUrl` in om Grok-webzoekopdrachten
via een operatorproxy of een xAI-compatibel Responses-eindpunt te routeren. OpenClaw
verzendt POST-verzoeken naar `<baseUrl>/responses` nadat afsluitende schuine strepen zijn verwijderd. `x_search`
valt terug op dezelfde `webSearch.baseUrl`, tenzij
`plugins.entries.xai.config.xSearch.baseUrl` is ingesteld.

## Gerelateerd

- [Overzicht van Web Search](/nl/tools/web) -- alle providers en automatische detectie
- [x_search in Web Search](/nl/tools/web#x_search) -- volwaardige zoekfunctie voor X via xAI
- [Gemini Search](/nl/tools/gemini-search) -- door AI samengestelde antwoorden via Google-onderbouwing
