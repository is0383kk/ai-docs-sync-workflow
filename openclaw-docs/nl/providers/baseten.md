---
read_when:
    - Je wilt Inkling van Thinking Machines Lab uitvoeren in OpenClaw
    - Je wilt één OpenAI-compatibele API voor de gehoste modellen van Baseten
summary: Baseten-configuratie voor Inkling en gehoste model-API's
title: Baseten
x-i18n:
    generated_at: "2026-07-27T06:30:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ccc3b5cf64b01859f9f022d7bc15a69a1cb42c87d4f914c118276c1151020de
    source_path: providers/baseten.md
    workflow: 16
---

[Baseten Model-API's](https://docs.baseten.co/inference/model-apis/overview) bieden gehoste, OpenAI-compatibele toegang tot geavanceerde modellen. De officiële externe Plugin gebruikt geauthenticeerde detectie, zodat OpenClaw de volledige modelset volgt die voor je Baseten-account is ingeschakeld. De offline fallback bevat elke Model-API die beschikbaar was toen deze OpenClaw-release werd gebouwd.

| Eigenschap       | Waarde                                                   |
| ---------------- | -------------------------------------------------------- |
| Provider-id      | `baseten`                                       |
| Plugin           | officieel extern pakket (`@openclaw/baseten-provider`)             |
| Auth-omgevingsvariabele | `BASETEN_API_KEY`                                |
| Onboarding-vlag  | `--auth-choice baseten-api-key`                                       |
| Directe CLI-vlag | `--baseten-api-key <key>`                                       |
| API              | OpenAI-compatibel (`openai-completions`)                   |
| Basis-URL        | `https://inference.baseten.co/v1`                                       |
| Standaardmodel   | `baseten/thinkingmachines/inkling`                                       |

## Plugin installeren

```bash
openclaw plugins install @openclaw/baseten-provider
openclaw gateway restart
```

## Aan de slag

<Steps>
  <Step title="Maak een Baseten-account en API-sleutel aan">
    Het Basic-abonnement van Baseten heeft geen maandelijkse platformkosten; Model-API-aanroepen worden op basis van gebruik geprijsd. Maak een sleutel aan in de [instellingen voor Baseten-API-sleutels](https://app.baseten.co/settings/api_keys) en bekijk de actuele tarieven op de [prijzenpagina](https://www.baseten.co/pricing).
  </Step>
  <Step title="Voer de onboarding uit">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice baseten-api-key
```

```bash Directe vlag
openclaw onboard --non-interactive \
  --auth-choice baseten-api-key \
  --baseten-api-key "$BASETEN_API_KEY"
```

```bash Alleen omgeving
export BASETEN_API_KEY=...
```

    </CodeGroup>

  </Step>
  <Step title="Controleer de livecatalogus">
    ```bash
    openclaw models list --provider baseten
    ```

    Met bruikbare authenticatie vraagt de Plugin `GET /v1/models` op en vermeldt deze elk model dat voor het account wordt geretourneerd. Zonder authenticatie blijft de Plugin offline en gebruikt deze de meegeleverde fallback.

  </Step>
</Steps>

## Inkling

[Inkling van Thinking Machines Lab](https://thinkingmachines.ai/news/introducing-inkling/) is het standaardmodel. In OpenClaw ondersteunt het tekst- en afbeeldingsinvoer, toolaanroepen, gestructureerde toolschema's, configureerbare redeneerinspanning, een contextvenster van 1.048M tokens en maximaal 32k uitvoertokens:

```json5
{
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
}
```

Gebruik `/model baseten/thinkingmachines/inkling` om een bestaande chat over te schakelen.

## Meegeleverde fallbackcatalogus

De geauthenticeerde livecatalogus is leidend. Deze rijen houden de configuratie en modelselectie bruikbaar voordat de detectie slaagt:

| Modelreferentie                                    | Invoer          | Context | Maximale uitvoer |
| -------------------------------------------------- | --------------- | ------: | ---------------: |
| `baseten/deepseek-ai/DeepSeek-V4-Pro`                                 | tekst           |    262k |             262k |
| `baseten/zai-org/GLM-4.7`                                 | tekst           |    200k |             200k |
| `baseten/zai-org/GLM-5`                                 | tekst           |    202k |             202k |
| `baseten/zai-org/GLM-5.1`                                 | tekst           |    202k |             202k |
| `baseten/zai-org/GLM-5.2`                                 | tekst           |    202k |             202k |
| `baseten/thinkingmachines/inkling`                                 | tekst, afbeelding |  1.048M |              32k |
| `baseten/moonshotai/Kimi-K2.5`                                 | tekst, afbeelding |    262k |             262k |
| `baseten/moonshotai/Kimi-K2.6`                                 | tekst, afbeelding |    262k |             262k |
| `baseten/moonshotai/Kimi-K2.7-Code`                                 | tekst, afbeelding |    262k |             262k |
| `baseten/nvidia/Nemotron-120B-A12B`                                 | tekst           |    202k |             202k |
| `baseten/nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B`                                 | tekst           |    202k |             202k |
| `baseten/openai/gpt-oss-120b`                                 | tekst           |    128k |             128k |

Alle meegeleverde modellen ondersteunen toolaanroepen en redeneren. OpenClaw koppelt zijn denkniveaus aan modellen met native `reasoning_effort`. De optionele GLM-, Kimi- en Nemotron-modellen van Baseten hebben denken standaard uitgeschakeld; de meeste bieden een binaire uit/aan-regeling, terwijl GLM 5.2 uit, hoog en maximaal biedt. OpenClaw verzendt deze keuzes via de `chat_template_args.enable_thinking`-regeling van Baseten en, voor GLM 5.2, de gevalideerde `reasoning_effort`-parameter op het hoogste niveau.

<Note>
Baseten kan onafhankelijk van OpenClaw-releases Model-API's toevoegen, verwijderen of wijzigen. De Plugin vernieuwt model-id's, contextlimieten, uitvoerlimieten en prijzen voor invoer, gecachte invoer en uitvoer via de geauthenticeerde API, terwijl het modelspecifieke OpenClaw-transportbeleid behouden blijft.
</Note>

## Handmatige configuratie

Voor de meeste configuraties is alleen de API-sleutel nodig. Om de provider expliciet vast te zetten:

```json5
{
  env: { BASETEN_API_KEY: "..." },
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      baseten: {
        baseUrl: "https://inference.baseten.co/v1",
        apiKey: "${BASETEN_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "thinkingmachines/inkling",
            name: "Inkling",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<Note>
Als de Gateway als daemon wordt uitgevoerd (launchd, systemd, Docker), zorg er dan voor dat `BASETEN_API_KEY` beschikbaar is voor dat proces. Een sleutel die alleen in een interactieve shell is geëxporteerd, is niet zichtbaar voor een reeds actieve beheerde service.
</Note>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Denkmodi" href="/nl/tools/thinking" icon="brain">
    Selecteer de niveaus voor redeneerinspanning van OpenClaw.
  </Card>
  <Card title="Modellen-CLI" href="/nl/cli/models" icon="terminal">
    Gedetecteerde modellen weergeven, inspecteren en selecteren.
  </Card>
  <Card title="Veelgestelde vragen over modellen" href="/nl/help/faq-models" icon="circle-question">
    Problemen met authenticatieprofielen en modelselectie oplossen.
  </Card>
</CardGroup>
