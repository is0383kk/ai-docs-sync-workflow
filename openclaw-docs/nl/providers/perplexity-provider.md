---
read_when:
    - Je wilt Perplexity configureren als provider voor zoeken op het web
    - Je hebt de Perplexity-API-sleutel of de OpenRouter-proxyconfiguratie nodig
summary: Configuratie van Perplexity als webzoekprovider (API-sleutel, zoekmodi, filtering)
title: Perplexity
x-i18n:
    generated_at: "2026-07-27T05:20:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea76a5cb7befce95756e9bcc8f9c1637fac87711d02d8a486ec2a1b9f51b73dc
    source_path: providers/perplexity-provider.md
    workflow: 16
---

De Perplexity-plugin registreert een `web_search`-provider met twee transportmethoden: de
native Perplexity Search API (gestructureerde resultaten met filters) en Perplexity
Sonar-chatvoltooiingen, rechtstreeks of via OpenRouter (door AI samengestelde antwoorden met
bronverwijzingen).

<Note>
Deze pagina behandelt de configuratie van de Perplexity-**provider**. Zie [Zoeken met Perplexity](/nl/tools/perplexity-search) voor de Perplexity-**tool** (hoe de agent deze gebruikt).
</Note>

| Eigenschap  | Waarde                                                                 |
| ----------- | ---------------------------------------------------------------------- |
| Type        | Webzoekprovider (geen modelprovider)                                   |
| Auth        | `PERPLEXITY_API_KEY` (native) of `OPENROUTER_API_KEY` (via OpenRouter) |
| Configuratiepad | `plugins.entries.perplexity.config.webSearch.apiKey`                   |
| Overschrijvingen | `plugins.entries.perplexity.config.webSearch.baseUrl` / `.model`       |
| Sleutel verkrijgen | [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)   |

## Plugin installeren

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## Aan de slag

<Steps>
  <Step title="Stel de API-sleutel in">
    ```bash
    openclaw configure --section web
    ```

    Of stel de sleutel rechtstreeks in:

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

    Een sleutel die als `PERPLEXITY_API_KEY` of `OPENROUTER_API_KEY` in de Gateway-omgeving
    is geëxporteerd, werkt ook.

  </Step>
  <Step title="Begin met zoeken">
    `web_search` detecteert Perplexity automatisch zodra de sleutel de beschikbare
    zoekreferentie is; verdere configuratie is niet nodig. Om de provider expliciet vast te zetten:

    ```bash
    openclaw config set tools.web.search.provider perplexity
    ```

  </Step>
</Steps>

## Zoekmodi

De Plugin bepaalt de transportmethode in deze volgorde:

1. `webSearch.baseUrl` of `webSearch.model` ingesteld: routeert altijd via Sonar-chatvoltooiingen naar dat eindpunt, ongeacht het sleuteltype.
2. Anders bepaalt de bron van de sleutel het eindpunt: het voorvoegsel van een geconfigureerde sleutel selecteert de transportmethode (configuratie heeft voorrang op omgevingsvariabelen); een omgevingssleutel gebruikt rechtstreeks het bijbehorende eindpunt.

| Sleutelvoorvoegsel | Transport                                                  | Functies                                         |
| ---------- | ---------------------------------------------------------- | ------------------------------------------------ |
| `pplx-`    | Native Perplexity Search API (`https://api.perplexity.ai`) | Gestructureerde resultaten, domein-/taal-/datumfilters |
| `sk-or-`   | OpenRouter (`https://openrouter.ai/api/v1`), Sonar-model   | Door AI samengestelde antwoorden met bronverwijzingen |

Een geconfigureerde sleutel met een ander voorvoegsel gebruikt eveneens de native Search API. Het
chatvoltooiingspad gebruikt standaard het `perplexity/sonar-pro`-model; overschrijf dit
met `plugins.entries.perplexity.config.webSearch.model`.

## Filters voor de native API

| Filter                               | Beschrijving                                                    | Transport   |
| ------------------------------------ | --------------------------------------------------------------- | ----------- |
| `count`                              | Resultaten per zoekopdracht, 1-10 (standaard 5)                 | Alleen native |
| `freshness`                          | Recentheidsperiode: `day`, `week`, `month`, `year`                  | Beide       |
| `country`                            | Landcode van 2 letters (`us`, `de`, `jp`)                        | Alleen native |
| `language`                           | ISO 639-1-taalcode (`en`, `fr`, `zh`)                      | Alleen native |
| `date_after` / `date_before`         | Publicatiedatumbereik in `YYYY-MM-DD`                            | Alleen native |
| `domain_filter`                      | Maximaal 20 domeinen; acceptatielijst of weigeringslijst met het voorvoegsel `-`, nooit gecombineerd | Alleen native |
| `max_tokens` / `max_tokens_per_page` | Inhoudsbudget voor alle resultaten / per pagina                 | Alleen native |

Filters die alleen native beschikbaar zijn, retourneren een beschrijvende fout in het chatvoltooiingspad.
`freshness` kan niet worden gecombineerd met `date_after`/`date_before`.

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Omgevingsvariabele voor daemonprocessen">
    <Warning>
    Een sleutel die alleen in een interactieve shell is geëxporteerd, is niet zichtbaar voor een
    launchd/systemd-Gateway-daemon, tenzij die omgeving expliciet wordt
    geïmporteerd. Stel de sleutel in `~/.openclaw/.env` of via `env.shellEnv` in, zodat het
    Gateway-proces deze kan lezen. Zie [Omgevingsvariabelen](/nl/help/environment)
    voor de volledige prioriteitsvolgorde.
    </Warning>
  </Accordion>

  <Accordion title="OpenRouter-proxy configureren">
    Om Perplexity-zoekopdrachten via OpenRouter te routeren, stel je een `OPENROUTER_API_KEY`
    (voorvoegsel `sk-or-`) in in plaats van een native Perplexity-sleutel. OpenClaw detecteert de
    sleutel en schakelt automatisch over naar de Sonar-transportmethode. Dit is handig als je al
    facturering via OpenRouter hebt ingesteld en providers daar wilt consolideren.
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Perplexity-zoektool" href="/nl/tools/perplexity-search" icon="magnifying-glass">
    Hoe de agent Perplexity-zoekopdrachten aanroept en resultaten interpreteert.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledige configuratiereferentie, inclusief Plugin-vermeldingen.
  </Card>
</CardGroup>
