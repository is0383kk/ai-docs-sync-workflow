---
read_when:
    - Je wilt Moonshot Kimi K3/K2 (Moonshot Open Platform) vergelijken met de configuratie van Kimi Coding
    - Je moet afzonderlijke endpoints, sleutels en modelreferenties begrijpen
    - Je wilt een configuratie die je voor beide providers kunt kopiëren en plakken
summary: Moonshot Kimi-modellen versus Kimi Coding configureren (afzonderlijke providers + sleutels)
title: Moonshot AI
x-i18n:
    generated_at: "2026-07-27T06:31:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 213379bf88fec26b052184a920e112f0887d6485601bfb47f590cf37ef983e58
    source_path: providers/moonshot.md
    workflow: 16
---

Moonshot biedt de Kimi-API met OpenAI-compatibele endpoints. Selecteer
`moonshot/kimi-k3` voor Kimi K3, behoud de standaardwaarde voor onboarding
`moonshot/kimi-k2.6`, of gebruik `kimi/kimi-for-coding` voor Kimi Coding.

<Warning>
Moonshot en Kimi Coding zijn **afzonderlijke providers**, die elk als afzonderlijke externe plugin worden geleverd. Sleutels zijn niet onderling uitwisselbaar, endpoints verschillen en modelreferenties verschillen (`moonshot/...` versus `kimi/...`).
</Warning>

## Ingebouwde modelcatalogus

[//]: # "moonshot-kimi-k2-ids:start"

| Modelreferentie                     | Naam                     | Redeneren     | Invoer          | Context   | Maximale uitvoer |
| ----------------------------------- | ------------------------ | ------------- | --------------- | --------- | ---------------- |
| `moonshot/kimi-k2.6`                  | Kimi K2.6                | Nee           | tekst, afbeelding | 262,144   | 262,144          |
| `moonshot/kimi-k3`                  | Kimi K3                  | Altijd maximaal | tekst, afbeelding | 1,048,576 | 1,048,576        |
| `moonshot/kimi-k2.7-code`                  | Kimi K2.7 Code           | Altijd aan    | tekst, afbeelding | 262,144   | 262,144          |
| `moonshot/kimi-k2.7-code-highspeed`                  | Kimi K2.7 Code HighSpeed | Altijd aan    | tekst, afbeelding | 262,144   | 262,144          |
| `moonshot/kimi-k2.5`                  | Kimi K2.5                | Nee           | tekst, afbeelding | 262,144   | 262,144          |

[//]: # "moonshot-kimi-k2-ids:end"

De kostenramingen in de catalogus gebruiken de gepubliceerde pay-as-you-go-tarieven van Moonshot. Controleer de
actuele leverancierspagina's voor [Kimi K3](https://platform.kimi.ai/docs/pricing/chat-k3),
[Kimi K2.7 Code](https://platform.kimi.ai/docs/pricing/chat-k27-code),
[Kimi K2.6](https://platform.kimi.ai/docs/pricing/chat-k26) en
[Kimi K2.5](https://platform.kimi.ai/docs/pricing/chat-k25) voordat je beslissingen
over kosten neemt.

Kimi K3 redeneert altijd op `reasoning_effort: "max"`. OpenClaw stelt alleen
`/think max` beschikbaar, laat het uitsluitend voor K2 bestemde veld `thinking` weg en verwijdert sampling-
overschrijvingen (`temperature`, `top_p`, `n`, `presence_penalty` en
`frequency_penalty`) die K3 vastzet op de standaardwaarden van de provider. Kimi K2.7 Code
gebruikt ook altijd native thinking, maar vereist dat zowel `thinking` als
`reasoning_effort` worden weggelaten; de HighSpeed-variant gebruikt hetzelfde contract.
Kimi K2.6 blijft de standaardwaarde voor onboarding.
Zie de [snelstartgids voor Kimi K3](https://platform.kimi.ai/docs/guide/kimi-k3-quickstart) van Moonshot.

## Aan de slag

Zowel Moonshot als Kimi Coding zijn externe plugins — installeer er één voordat je
de onboarding uitvoert.

<Tabs>
  <Tab title="Moonshot API">
    **Het meest geschikt voor:** Kimi K3- en K2-modellen via het Moonshot Open Platform.

    <Steps>
      <Step title="Installeer de plugin">
        ```bash
        openclaw plugins install @openclaw/moonshot-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="Kies de regio van je endpoint">
        | Authenticatiekeuze     | Endpoint                       | Regio         |
        | ---------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`             | Internationaal |
        | `moonshot-api-key-cn`     | `https://api.moonshot.cn/v1`             | China         |
      </Step>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        Of voor het Chinese endpoint:

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```
      </Step>
      <Step title="Stel Kimi K3 in als standaardmodel">
        Tijdens de onboarding blijft Kimi K2.6 de oorspronkelijke standaardwaarde. Schakel expliciet over
        wanneer je Kimi K3 wilt gebruiken:

        ```bash
        openclaw models set moonshot/kimi-k3
        ```
      </Step>
      <Step title="Controleer of de modellen beschikbaar zijn">
        ```bash
        openclaw models list --provider moonshot
        ```
      </Step>
      <Step title="Voer een live-rooktest uit">
        Gebruik een geïsoleerde statusmap wanneer je modeltoegang en kostenregistratie wilt verifiëren
        zonder je normale sessies te wijzigen:

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message 'Reply exactly: KIMI_LIVE_OK' \
          --thinking max \
          --json
        ```

        Het JSON-antwoord moet `provider: "moonshot"` en
        `model: "kimi-k3"` rapporteren. Het transcriptitem van de assistent bewaart genormaliseerd
        tokengebruik plus de geraamde kosten onder `usage.cost` wanneer Moonshot
        gebruiksmetadata retourneert.
      </Step>
    </Steps>

    ### Configuratievoorbeeld

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k3": { alias: "Kimi K3" },
            "moonshot/kimi-k2.7-code": { alias: "Kimi K2.7 Code" },
            "moonshot/kimi-k2.7-code-highspeed": { alias: "Kimi K2.7 Code HighSpeed" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k3",
                name: "Kimi K3",
                reasoning: true,
                thinkingLevelMap: {
                  off: null,
                  minimal: null,
                  low: null,
                  medium: null,
                  high: null,
                  xhigh: "max",
                  max: "max",
                },
                input: ["text", "image"],
                cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0 },
                contextWindow: 1048576,
                maxTokens: 1048576,
              },
              {
                id: "kimi-k2.7-code",
                name: "Kimi K2.7 Code",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.19, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.7-code-highspeed",
                name: "Kimi K2.7 Code HighSpeed",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 1.9, output: 8, cacheRead: 0.38, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

  </Tab>

  <Tab title="Kimi Coding">
    **Het meest geschikt voor:** codegerichte taken via het Kimi Coding-endpoint.

    <Note>
    Kimi Coding gebruikt een andere API-sleutel en providerprefix (`kimi/...`) dan Moonshot (`moonshot/...`). De huidige referenties zijn `kimi/k3` voor een context van 256K, `kimi/k3[1m]` voor de 1M-laag, `kimi/kimi-for-coding` en `kimi/kimi-for-coding-highspeed`. Verouderde referenties `kimi/kimi-code` en `kimi/k2p5` blijven geaccepteerd en worden genormaliseerd naar `kimi/kimi-for-coding`.
    </Note>

    De programmeerservice accepteert zowel OpenAI-compatibele
    `https://api.kimi.com/coding/v1`-clients als Anthropic-compatibele
    `https://api.kimi.com/coding/`-clients. Deze plugin gebruikt Anthropic Messages.
    Maak lidmaatschapssleutels aan in de
    [Kimi Code Console](https://www.kimi.com/code/console); de huidige lidmaatschapsprijzen
    staan op [de prijzenpagina van Kimi](https://www.kimi.com/membership/pricing).

    <Steps>
      <Step title="Installeer de plugin">
        ```bash
        openclaw plugins install @openclaw/kimi-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```
      </Step>
      <Step title="Stel een standaardmodel in">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-for-coding" },
            },
          },
        }
        ```
      </Step>
      <Step title="Controleer of het model beschikbaar is">
        ```bash
        openclaw models list --provider kimi
        ```
      </Step>
    </Steps>

    Kimi Code K3 gebruikt standaard deep thinking op `max`. `/think off` verzendt
    `thinking.type: "disabled"`; `/think max` verzendt het adaptive-thinking-
    verzoek van K3 met maximale inspanning. Verouderde lagere thinking-niveaus worden omgezet naar het
    ondersteunde niveau `max`. Het 1M-model vereist een Allegretto- of hoger Kimi-
    lidmaatschap; gebruik `kimi/k3` bij Moderato.

    Zie de officiële [Kimi Code-modeltabel](https://www.kimi.com/code/docs/en/kimi-code/models.html) voor de huidige beschikbaarheid per abonnement.

    ### Configuratievoorbeeld

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: {
            "kimi/kimi-for-coding": { alias: "Kimi" },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## Kimi-webzoekfunctie

De Moonshot-plugin registreert ook **Kimi** als een `web_search`-provider, ondersteund door Moonshot-webzoekopdrachten.

<Steps>
  <Step title="Voer de interactieve configuratie voor webzoekopdrachten uit">
    ```bash
    openclaw configure --section web
    ```

    Kies **Kimi** in de sectie voor webzoekopdrachten om
    `plugins.entries.moonshot.config.webSearch.*` op te slaan.

  </Step>
  <Step title="Configureer de regio en het model voor webzoekopdrachten">
    De interactieve configuratie vraagt om:

    | Instelling          | Opties                                                               |
    | ------------------- | -------------------------------------------------------------------- |
    | API-regio           | `https://api.moonshot.ai/v1` (internationaal) of `https://api.moonshot.cn/v1` (China) |
    | Webzoekmodel        | Standaardwaarde is `kimi-k2.6`                                |

  </Step>
</Steps>

De configuratie staat onder `plugins.entries.moonshot.config.webSearch`:

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // or use KIMI_API_KEY / MOONSHOT_API_KEY
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

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Native thinking-modus">
    Moonshot API Kimi K3 redeneert altijd met maximale inspanning. OpenClaw stelt alleen
    `/think max` beschikbaar, verzendt `reasoning_effort: "max"` en negeert verouderde lagere of
    `off`-instellingen.

    Kimi Code K3 biedt `/think off|max`. Het Anthropic-compatibele eindpunt
    ontvangt `thinking.type: "disabled"` voor uitgeschakeld, of adaptief denken met
    `output_config.effort: "max"` voor maximaal. Dit geldt voor zowel `kimi/k3` als
    `kimi/k3[1m]`.
    Moonshot API K3 ondersteunt `auto`, `none`, `required` en vastgezette toolkeuzes,
    zodat OpenClaw de aangevraagde `tool_choice` behoudt. Bij toolgebruik over meerdere beurten
    behoudt OpenClaw de redeneerinhoud van de assistent die vereist is voor het
    replaycontract van Moonshot.

    Kimi K2.7 Code gebruikt altijd systeemeigen denken. Moonshot vereist dat clients
    het veld `thinking` voor dit model weglaten, zodat OpenClaw alleen `on` aanbiedt en
    verouderde instellingen voor `off` negeert. K2.7 stelt ook `temperature`, `top_p`, `n`,
    `presence_penalty` en `frequency_penalty` vast; OpenClaw laat geconfigureerde
    overschrijvingen voor die velden weg.

    Andere Moonshot Kimi-modellen ondersteunen binair systeemeigen denken:

    - `thinking: { type: "enabled" }`
    - `thinking: { type: "disabled" }`

    Configureer dit per model via `agents.defaults.models.<provider/model>.params`:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "disabled" },
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw wijst runtime-niveaus voor `/think` voor die modellen als volgt toe:

    | Niveau van `/think`       | Moonshot-gedrag          |
    | -------------------- | -------------------------- |
    | `/think off`         | `thinking.type=disabled`   |
    | Elk niveau behalve uitgeschakeld    | `thinking.type=enabled`    |

    <Warning>
    Wanneer denken voor Moonshot K2 is ingeschakeld, moet `tool_choice` `auto` of `none` zijn. Een vastgezette toolkeuze (`type: "tool"` of `type: "function"`) dwingt denken in plaats daarvan terug naar `disabled`, zodat de aangevraagde tool alsnog wordt uitgevoerd; `tool_choice: "required"` wordt in plaats daarvan genormaliseerd naar `auto`. Kimi K2.7 Code kan denken niet uitschakelen, zodat de incompatibele `tool_choice` wordt genormaliseerd naar `auto`. Kimi K3 gebruikt een afzonderlijk contract voor redeneerinspanning en behoudt ondersteunde toolkeuzes.
    </Warning>

    Kimi K2.6 accepteert ook een optioneel veld `thinking.keep` dat het
    behoud van `reasoning_content` over meerdere beurten regelt. Stel dit in op `"all"` om de volledige
    redenering tussen beurten te behouden; laat het weg (of laat het op `null`) om de
    standaardstrategie van de server te gebruiken. OpenClaw stuurt `thinking.keep` alleen door voor
    `moonshot/kimi-k2.6` en verwijdert het uit andere modellen. Kimi K2.7 Code
    behoudt standaard de volledige redeneergeschiedenis, terwijl OpenClaw het volledige
    veld `thinking` weglaat.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "enabled", keep: "all" },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Opschoning van toolaanroep-id's">
    Moonshot Kimi levert systeemeigen tool_call-id's met een vorm zoals `functions.<name>:<index>`. OpenClaw behoudt het eerste voorkomen van elke systeemeigen Kimi-id en herschrijft latere duplicaten naar deterministische `call_*`-id's in OpenAI-stijl. Overeenkomende toolresultaten worden opnieuw toegewezen met dezelfde id, zodat de replay uniek blijft zonder Kimi's eerste systeemeigen id te verwijderen. Dit gedrag is ingebouwd in de gebundelde Moonshot-provider en is geen door de gebruiker configureerbare instelling.
  </Accordion>

  <Accordion title="Compatibiliteit van streaminggebruik">
    Systeemeigen Moonshot-eindpunten (`https://api.moonshot.ai/v1` en
    `https://api.moonshot.cn/v1`) geven compatibiliteit met streaminggebruik aan.
    OpenClaw baseert dit op de host van het eindpunt, niet op de provider-id, zodat een aangepaste
    provider-id die naar dezelfde systeemeigen Moonshot-host verwijst, hetzelfde
    gedrag voor streaminggebruik overneemt.

    Met de K2.6-prijzen uit de catalogus wordt gestreamd gebruik dat invoer-, uitvoer-
    en cacheleestokens bevat ook omgezet in lokaal geschatte kosten in USD voor
    `/status`, `/usage full`, `/usage cost` en sessieboekhouding
    op basis van transcripties.

  </Accordion>

  <Accordion title="Naslag voor eindpunten en modelverwijzingen">
    | Provider   | Voorvoegsel modelverwijzing | Eindpunt                      | Omgevingsvariabele voor authenticatie        |
    | ---------- | ---------------- | ------------------------------ | ------------------- |
    | Moonshot   | `moonshot/`      | `https://api.moonshot.ai/v1`  | `MOONSHOT_API_KEY`  |
    | Moonshot CN| `moonshot/`      | `https://api.moonshot.cn/v1`  | `MOONSHOT_API_KEY`  |
    | Kimi Coding| `kimi/`          | Kimi Coding-eindpunt           | `KIMI_API_KEY`      |
    | Zoeken op het web | N.v.t.              | Hetzelfde als de Moonshot API-regio    | `KIMI_API_KEY` of `MOONSHOT_API_KEY` |

    - Kimi-webzoekopdrachten gebruiken `KIMI_API_KEY` of `MOONSHOT_API_KEY` en gebruiken standaard `https://api.moonshot.ai/v1` met model `kimi-k2.6`.
    - Overschrijf indien nodig prijs- en contextmetadata in `models.providers`.
    - Als Moonshot andere contextlimieten voor een model publiceert, pas `contextWindow` dan dienovereenkomstig aan.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelverwijzingen en failovergedrag kiezen.
  </Card>
  <Card title="Zoeken op het web" href="/nl/tools/web" icon="magnifying-glass">
    Providers voor zoeken op het web configureren, waaronder Kimi.
  </Card>
  <Card title="Configuratienaslag" href="/nl/gateway/configuration-reference" icon="gear">
    Volledig configuratieschema voor providers, modellen en plugins.
  </Card>
  <Card title="Moonshot Open Platform" href="https://platform.moonshot.ai" icon="globe">
    Beheer en documentatie van Moonshot API-sleutels.
  </Card>
</CardGroup>
