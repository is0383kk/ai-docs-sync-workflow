---
read_when:
    - Je wilt toegang tot modellen die door OpenCode worden gehost
    - Je wilt kiezen tussen de Zen- en Go-catalogi
summary: Gebruik OpenCode Zen- en Go-catalogi met OpenClaw
title: OpenCode
x-i18n:
    generated_at: "2026-07-27T06:31:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de287eb8a349f26c265f95b8b1de3af4035aa2bdc3501c7279f714d297bb8b9b
    source_path: providers/opencode.md
    workflow: 16
---

OpenCode biedt twee gehoste catalogi in OpenClaw:

| Catalogus | Voorvoegsel            | Runtimeprovider |
| --------- | ---------------------- | --------------- |
| **Zen**   | `opencode/...`     | `opencode` |
| **Go**    | `opencode-go/...`     | `opencode-go` |

Beide catalogi delen één OpenCode-API-sleutel (`OPENCODE_API_KEY`, alias
`OPENCODE_ZEN_API_KEY`). OpenClaw houdt de runtimeprovider-id's gescheiden, zodat
de upstreamroutering per model correct blijft, maar behandelt ze tijdens de
onboarding en in de documentatie als één OpenCode-configuratie.

## Aan de slag

<Tabs>
  <Tab title="Zen-catalogus">
    **Het meest geschikt voor:** de beheerde OpenCode-proxy voor meerdere modellen (Claude, GPT, Gemini, GLM,
    DeepSeek, Kimi, MiniMax, Qwen).

    <Steps>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard --auth-choice opencode-zen
        ```

        Of geef de sleutel rechtstreeks door:

        ```bash
        openclaw onboard --opencode-zen-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Stel een Zen-model als standaardmodel in">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode/claude-opus-4-6"
        ```
      </Step>
      <Step title="Controleer of modellen beschikbaar zijn">
        ```bash
        openclaw models list --provider opencode
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Go-catalogus">
    **Het meest geschikt voor:** het door OpenCode gehoste aanbod van Kimi, GLM, MiniMax, Qwen en DeepSeek.

    <Steps>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard --auth-choice opencode-go
        ```

        Of geef de sleutel rechtstreeks door:

        ```bash
        openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Stel een Go-model als standaardmodel in">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode-go/kimi-k2.6"
        ```
      </Step>
      <Step title="Controleer of modellen beschikbaar zijn">
        ```bash
        openclaw models list --provider opencode-go
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Configuratievoorbeeld

```json5
{
  env: { OPENCODE_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

## Ingebouwde catalogi

### Zen

| Eigenschap       | Waarde                                                                                        |
| ---------------- | --------------------------------------------------------------------------------------------- |
| Runtimeprovider  | `opencode`                                                                            |
| Voorbeeldmodellen | `opencode/claude-opus-4-6`, `opencode/gpt-5.5`, `opencode/gemini-3.1-pro`, `opencode/glm-5.2` |

Voer `openclaw models list --provider opencode` uit voor de volledige actuele lijst, die
ook rijen uit het gratis niveau bevat, zoals `opencode/big-pickle` en
`opencode/deepseek-v4-flash-free`.

### Go

| Eigenschap       | Waarde                                                                   |
| ---------------- | ------------------------------------------------------------------------ |
| Runtimeprovider  | `opencode-go`                                                       |
| Voorbeeldmodellen | `opencode-go/kimi-k2.6`, `opencode-go/glm-5`, `opencode-go/minimax-m2.5` |

Zie [OpenCode Go](/nl/providers/opencode-go) voor de volledige tabel met Go-modellen.

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Aliassen voor API-sleutels">
    `OPENCODE_ZEN_API_KEY` wordt ook geaccepteerd als alias voor `OPENCODE_API_KEY`.
  </Accordion>

  <Accordion title="Gedeelde aanmeldgegevens">
    Als je tijdens de configuratie één OpenCode-sleutel invoert, worden de
    aanmeldgegevens voor beide runtimeproviders opgeslagen. Je hoeft de onboarding
    niet voor elke catalogus afzonderlijk uit te voeren.
  </Accordion>

  <Accordion title="Een API-sleutel verkrijgen">
    Maak een OpenCode-account en genereer een API-sleutel op
    [opencode.ai/auth](https://opencode.ai/auth). Facturering en de
    beschikbaarheid van catalogi worden beheerd via het OpenCode-dashboard.
  </Accordion>

  <Accordion title="Replaygedrag van Gemini">
    Door Gemini ondersteunde OpenCode-referenties blijven het proxy-Gemini-pad gebruiken, zodat OpenClaw
    daar de opschoning van Gemini-denkhandtekeningen behoudt zonder systeemeigen
    replayvalidatie of bootstrap-herschrijvingen van Gemini in te schakelen.
  </Accordion>

  <Accordion title="Replaygedrag van niet-Gemini-modellen">
    OpenCode-referenties die niet van Gemini zijn, behouden het minimale OpenAI-compatibele replaybeleid.
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="OpenCode Go" href="/nl/providers/opencode-go" icon="server">
    Volledige referentie voor de Go-catalogus.
  </Card>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledige configuratiereferentie voor agents, modellen en providers.
  </Card>
</CardGroup>
