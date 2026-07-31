---
read_when:
    - Je wilt Chutes met OpenClaw gebruiken
    - Je hebt het configuratiepad voor OAuth of de API-sleutel nodig
    - Je wilt het standaardmodel, aliassen of detectiegedrag
summary: Chutes-configuratie (OAuth of API-sleutel, modeldetectie, aliassen)
title: Chutes
x-i18n:
    generated_at: "2026-07-27T05:46:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 57ea5112105f19028c1a348b4d7fec4cf7ef12de00b1b2de9c152057bf5033a9
    source_path: providers/chutes.md
    workflow: 16
---

[Chutes](https://chutes.ai) stelt opensourcemodelcatalogi beschikbaar via een
OpenAI-compatibele API. OpenClaw ondersteunt zowel browser-OAuth als authenticatie met een API-sleutel.

| Eigenschap       | Waarde                                                  |
| ---------------- | ------------------------------------------------------- |
| Provider         | `chutes`                                      |
| Plugin           | officieel extern pakket (`@openclaw/chutes-provider`)            |
| API              | OpenAI-compatibel                                       |
| Basis-URL        | `https://llm.chutes.ai/v1`                                      |
| Authenticatie    | OAuth of API-sleutel (zie hieronder)                    |
| Runtime-omgevingsvariabelen | `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`        |

`CHUTES_OAUTH_TOKEN` geeft rechtstreeks een al verkregen OAuth-toegangstoken door
(bijvoorbeeld in CI), waarbij de onderstaande interactieve browserflow wordt overgeslagen.

## Plugin installeren

```bash
openclaw plugins install @openclaw/chutes-provider
openclaw gateway restart
```

## Aan de slag

Beide methoden stellen het standaardmodel in op `chutes/zai-org/GLM-5-TEE` en registreren
de Chutes-catalogus.

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="Voer de OAuth-onboardingflow uit">
        ```bash
        openclaw onboard --auth-choice chutes
        ```
        OpenClaw start de browserflow lokaal of toont op externe/headless hosts
        een URL met een flow waarin de omleiding wordt geplakt. OAuth-tokens worden automatisch vernieuwd via
        OpenClaw-authenticatieprofielen.
      </Step>
    </Steps>
  </Tab>
  <Tab title="API-sleutel">
    <Steps>
      <Step title="Verkrijg een API-sleutel">
        Maak een sleutel aan op
        [chutes.ai/settings/api-keys](https://chutes.ai/settings/api-keys).
      </Step>
      <Step title="Voer de onboardingflow voor de API-sleutel uit">
        ```bash
        openclaw onboard --auth-choice chutes-api-key
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## Detectiegedrag

Wanneer Chutes-authenticatie beschikbaar is, bevraagt OpenClaw `GET /v1/models` met die
referentie en gebruikt het de gedetecteerde modellen, die per referentie 5 minuten in de cache
blijven. Bij een verlopen/niet-geautoriseerde sleutel (HTTP 401) probeert OpenClaw het één keer opnieuw
zonder referenties. Als de detectie nog steeds geen rijen retourneert, mislukt of een
andere niet-2xx-status retourneert, valt OpenClaw terug op de meegeleverde statische catalogus (zowel detectie met
een API-sleutel als OAuth-detectie gebruikt hetzelfde pad). Als de detectie bij het opstarten mislukt, wordt de
statische catalogus automatisch gebruikt.

## Standaardaliassen

OpenClaw registreert twee handige aliassen voor de Chutes-catalogus:

| Alias           | Doelmodel                              |
| --------------- | -------------------------------------- |
| `chutes-pro`    | `chutes/deepseek-ai/DeepSeek-V3.2-TEE` |
| `chutes-vision` | `chutes/moonshotai/Kimi-K2.5-TEE`      |

## Ingebouwde startercatalogus

De meegeleverde terugvalcatalogus bevat deze vijf modellen die momenteel worden aangeboden:

| Modelreferentie                        |
| -------------------------------------- |
| `chutes/zai-org/GLM-5-TEE`             |
| `chutes/deepseek-ai/DeepSeek-V3.2-TEE` |
| `chutes/moonshotai/Kimi-K2.5-TEE`      |
| `chutes/MiniMaxAI/MiniMax-M2.5-TEE`    |
| `chutes/Qwen/Qwen3.5-397B-A17B-TEE`    |

Voer `openclaw models list --all --provider chutes` uit voor de volledige lijst.

## Configuratievoorbeeld

```json5
{
  agents: {
    defaults: {
      model: { primary: "chutes/zai-org/GLM-5-TEE" },
      models: {
        "chutes/zai-org/GLM-5-TEE": { alias: "Chutes GLM 5" },
        "chutes/deepseek-ai/DeepSeek-V3.2-TEE": { alias: "Chutes DeepSeek V3.2" },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="OAuth-overschrijvingen">
    Pas de OAuth-flow aan met optionele omgevingsvariabelen:

    | Variabele | Doel |
    | -------- | ------- |
    | `CHUTES_CLIENT_ID` | OAuth-client-id (er wordt om gevraagd indien niet ingesteld) |
    | `CHUTES_CLIENT_SECRET` | OAuth-clientgeheim |
    | `CHUTES_OAUTH_REDIRECT_URI` | Omleidings-URI (standaard `http://127.0.0.1:1456/oauth-callback`) |
    | `CHUTES_OAUTH_SCOPES` | Door spaties gescheiden bereiken (standaard `openid profile chutes:invoke`) |

    Zie de [Chutes-documentatie over OAuth](https://chutes.ai/docs/sign-in-with-chutes/overview)
    voor vereisten voor omleidingsapps en hulp.

  </Accordion>

  <Accordion title="Opmerkingen">
    - Chutes-modellen worden geregistreerd als `chutes/<model-id>`.
    - Chutes rapporteert tijdens het streamen geen tokengebruik (`supportsUsageInStreaming: false`); de gebruikstotalen worden alsnog weergegeven zodra de stream is voltooid.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providerregels, modelreferenties en failovergedrag.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledig configuratieschema, inclusief providerinstellingen.
  </Card>
  <Card title="Chutes" href="https://chutes.ai" icon="arrow-up-right-from-square">
    Chutes-dashboard en API-documentatie.
  </Card>
  <Card title="Chutes-API-sleutels" href="https://chutes.ai/settings/api-keys" icon="key">
    Maak Chutes-API-sleutels aan en beheer ze.
  </Card>
</CardGroup>
