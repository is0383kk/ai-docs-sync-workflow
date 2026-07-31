---
read_when:
    - Je wilt Cerebras gebruiken met OpenClaw
    - Je hebt de omgevingsvariabele voor de Cerebras-API-sleutel of de CLI-authenticatiekeuze nodig
summary: Cerebras-installatie (authenticatie + modelselectie)
title: Cerebras
x-i18n:
    generated_at: "2026-07-27T05:29:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 716eef83155ef80d9aa61bd55ed83e3e38ad22720ae055bce7eb9c2cbfb6cf41
    source_path: providers/cerebras.md
    workflow: 16
---

[Cerebras](https://www.cerebras.ai) biedt snelle, met OpenAI compatibele inferentie op aangepaste inferentiehardware. De Plugin wordt geleverd met een statische catalogus van twee modellen (geen live-detectie).

| Eigenschap      | Waarde                                                    |
| --------------- | --------------------------------------------------------- |
| Provider-id     | `cerebras`                                        |
| Plugin          | officieel extern pakket (`@openclaw/cerebras-provider`)              |
| Auth-omgevingsvariabele | `CEREBRAS_API_KEY`                               |
| Onboarding-vlag | `--auth-choice cerebras-api-key`                                        |
| Directe CLI-vlag | `--cerebras-api-key <key>`                                       |
| API             | met OpenAI compatibel (`openai-completions`)                |
| Basis-URL       | `https://api.cerebras.ai/v1`                                        |
| Standaardmodel  | `cerebras/zai-glm-4.7`                                        |

## Plugin installeren

```bash
openclaw plugins install @openclaw/cerebras-provider
openclaw gateway restart
```

## Aan de slag

<Steps>
  <Step title="Een API-sleutel verkrijgen">
    Maak een API-sleutel aan in de [Cerebras Cloud Console](https://cloud.cerebras.ai).
  </Step>
  <Step title="Onboarding uitvoeren">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice cerebras-api-key
```

```bash Directe vlag
openclaw onboard --non-interactive \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

```bash Alleen omgevingsvariabele
export CEREBRAS_API_KEY=csk-...
```

    </CodeGroup>

  </Step>
  <Step title="Controleren of modellen beschikbaar zijn">
    ```bash
    openclaw models list --provider cerebras
    ```

    Geeft beide statische modellen weer. Als `CEREBRAS_API_KEY` niet kan worden herleid, meldt `openclaw models status --json` de ontbrekende referentie onder `auth.unusableProfiles`.

  </Step>
</Steps>

## Niet-interactieve configuratie

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

## Ingebouwde catalogus

Beide modellen hebben een contextvenster van 128k en maximaal 8.192 uitvoertokens.

| Modelreferentie         | Naam         | Redeneren | Opmerkingen                              |
| ----------------------- | ------------ | --------- | ---------------------------------------- |
| `cerebras/zai-glm-4.7`      | Z.ai GLM 4.7 | ja        | Standaardmodel; preview-redeneermodel     |
| `cerebras/gpt-oss-120b`      | GPT OSS 120B | ja        | Redeneermodel voor productie              |

## Handmatige configuratie

Voor de meeste configuraties is alleen de API-sleutel nodig. Gebruik expliciete `models.providers.cerebras`-configuratie om modelmetadata te overschrijven of in `mode: "merge"` met de statische catalogus te werken:

```json5
{
  env: { CEREBRAS_API_KEY: "csk-..." },
  agents: {
    defaults: {
      model: { primary: "cerebras/zai-glm-4.7" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "Z.ai GLM 4.7" },
          { id: "gpt-oss-120b", name: "GPT OSS 120B" },
        ],
      },
    },
  },
}
```

<Note>
Als de Gateway als daemon wordt uitgevoerd (launchd, systemd, Docker), zorg er dan voor dat `CEREBRAS_API_KEY` beschikbaar is voor dat proces — bijvoorbeeld in `~/.openclaw/.env` of via `env.shellEnv`. Een sleutel die alleen in een interactieve shell is geëxporteerd, helpt een beheerde service niet, tenzij de omgevingsvariabele afzonderlijk wordt geïmporteerd.
</Note>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Denkmodi" href="/nl/tools/thinking" icon="brain">
    Niveaus voor redeneerinspanning voor de twee Cerebras-modellen die kunnen redeneren.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults" icon="gear">
    Standaardinstellingen voor agents en modelconfiguratie.
  </Card>
  <Card title="Veelgestelde vragen over modellen" href="/nl/help/faq-models" icon="circle-question">
    Auth-profielen, van model wisselen en fouten met "no profile" oplossen.
  </Card>
</CardGroup>
