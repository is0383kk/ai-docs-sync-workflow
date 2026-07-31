---
read_when:
    - Je wilt Synthetic als modelprovider gebruiken
    - Je moet een Synthetic-API-sleutel of basis-URL instellen
summary: Gebruik de Anthropic-compatibele API van Synthetic in OpenClaw
title: Synthetic
x-i18n:
    generated_at: "2026-07-27T05:31:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f6cc89a7b837f57555d176ce78e62a39095d4ef0765c96b6b7b93ffebd7388
    source_path: providers/synthetic.md
    workflow: 16
---

[Synthetic](https://synthetic.new) biedt Anthropic-compatibele eindpunten.
OpenClaw bundelt het als de provider `synthetic` en gebruikt de Anthropic
Messages-API.

| Eigenschap | Waarde                                |
| ---------- | ------------------------------------- |
| Provider   | `synthetic`                    |
| Auth       | `SYNTHETIC_API_KEY`                    |
| API        | Anthropic Messages                    |
| Basis-URL  | `https://api.synthetic.new/anthropic`                    |

## Aan de slag

<Steps>
  <Step title="Een API-sleutel verkrijgen">
    Haal een `SYNTHETIC_API_KEY` op uit je Synthetic-account of laat onboarding
    je erom vragen.
  </Step>
  <Step title="Onboarding uitvoeren">
    ```bash
    openclaw onboard --auth-choice synthetic-api-key
    ```
  </Step>
  <Step title="Het standaardmodel verifiëren">
    Onboarding stelt het standaardmodel in op:
    ```text
    synthetic/hf:MiniMaxAI/MiniMax-M3
    ```
  </Step>
</Steps>

<Warning>
De Anthropic-client van OpenClaw voegt automatisch `/v1` toe aan de basis-URL, dus gebruik
`https://api.synthetic.new/anthropic` (niet `/anthropic/v1`). Als Synthetic
de basis-URL wijzigt, overschrijf dan `models.providers.synthetic.baseUrl`.
</Warning>

## Configuratievoorbeeld

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M3",
            name: "MiniMax M3",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## Ingebouwde catalogus

Alle Synthetic-modellen gebruiken kosten `0` (invoer/uitvoer/cache). Bekijk de
[actuele modellenlijst](https://dev.synthetic.new/docs/api/models) van Synthetic voor de beschikbaarheid van de service.

| Model-ID                                            | Contextvenster | Max. tokens | Redeneren | Invoer           |
| --------------------------------------------------- | -------------- | ----------- | --------- | ---------------- |
| `hf:MiniMaxAI/MiniMax-M3`                                  | 262,144        | 65,536      | ja        | tekst + afbeelding |
| `hf:moonshotai/Kimi-K2.7-Code`                                  | 262,144        | 8,192       | ja        | tekst + afbeelding |
| `hf:nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4`                                  | 262,144        | 8,192       | ja        | tekst            |
| `hf:openai/gpt-oss-120b`                                  | 131,072        | 8,192       | ja        | tekst            |
| `hf:Qwen/Qwen3.6-27B`                                  | 262,144        | 81,920      | ja        | tekst + afbeelding |
| `hf:zai-org/GLM-4.7-Flash`                                  | 196,608        | 131,072     | ja        | tekst            |
| `hf:zai-org/GLM-5.2`                                  | 524,288        | 131,072     | ja        | tekst            |

<Tip>
Modelverwijzingen gebruiken de vorm `synthetic/<modelId>`. Gebruik
`openclaw models list --provider synthetic` om alle modellen te bekijken die beschikbaar zijn voor je
account.
</Tip>

<AccordionGroup>
  <Accordion title="Toegestane modellen">
    Als je een lijst met toegestane modellen inschakelt (`agents.defaults.modelPolicy.allow`), voeg dan elk
    Synthetic-model toe dat je wilt gebruiken. Modellen die niet in de lijst staan, zijn verborgen
    voor de agent.
  </Accordion>

  <Accordion title="Basis-URL overschrijven">
    Als Synthetic het API-eindpunt wijzigt, overschrijf dan de basis-URL:

    ```json5
    {
      models: {
        providers: {
          synthetic: {
            baseUrl: "https://new-api.synthetic.new/anthropic",
          },
        },
      },
    }
    ```

    OpenClaw voegt nog steeds automatisch `/v1` toe.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Providerregels, modelverwijzingen en failovergedrag.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledig configuratieschema, inclusief providerinstellingen.
  </Card>
  <Card title="Synthetic" href="https://synthetic.new" icon="arrow-up-right-from-square">
    Synthetic-dashboard en API-documentatie.
  </Card>
</CardGroup>
