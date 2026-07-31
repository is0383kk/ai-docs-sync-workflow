---
read_when:
    - Je wilt OpenClaw uitvoeren met modellen van GMI Cloud
    - Je hebt de provider-id, sleutel of het eindpunt van GMI nodig
summary: Gebruik de OpenAI-compatibele API van GMI Cloud met OpenClaw
title: GMI Cloud
x-i18n:
    generated_at: "2026-07-27T06:05:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21fd2a997f44e1f78d97a0fba24ca2bbc00dd193323da712d650ed4ba105355
    source_path: providers/gmi.md
    workflow: 16
---

GMI Cloud is een gehost inferentieplatform voor geavanceerde modellen en modellen met open gewichten
achter een OpenAI-compatibele API. In OpenClaw is het een officiële externe provider-
Plugin: installeer deze eenmaal, sla inloggegevens op via normale modelauthenticatie en gebruik
modelverwijzingen zoals `gmi/google/gemini-3.1-flash-lite`.

Gebruik GMI wanneer je één API-sleutel wilt voor meerdere gehoste modelfamilies, waaronder
Anthropic, DeepSeek, Google, Moonshot, OpenAI en Z.AI-routes die via de catalogus van GMI
beschikbaar zijn. Het werkt als secundaire provider voor model-fallback, voor het vergelijken van
gehoste routes van verschillende leveranciers, of wanneer een model bij GMI beschikbaar is voordat
je primaire provider het aanbiedt. OpenClaw beheert de provider-id, het authenticatieprofiel, de aliassen,
de initiële modelcatalogus en de basis-URL; GMI beheert de actuele modelbeschikbaarheid, facturering,
snelheidslimieten en al het routeringsbeleid aan providerzijde.

| Eigenschap    | Waarde                                   |
| ------------- | ---------------------------------------- |
| Provider-id   | `gmi` (aliassen: `gmi-cloud`, `gmicloud`) |
| Pakket        | `@openclaw/gmi-provider`                 |
| Omgevingsvariabele voor authenticatie | `GMI_API_KEY`                            |
| API           | OpenAI-compatibel (`openai-completions`) |
| Basis-URL     | `https://api.gmi-serving.com/v1`         |
| Standaardmodel | `gmi/google/gemini-3.1-flash-lite`       |

## Configuratie

Installeer de Plugin, start de Gateway opnieuw en maak vervolgens een API-sleutel aan in GMI Cloud
(`https://www.gmicloud.ai/`):

```bash
openclaw plugins install @openclaw/gmi-provider
openclaw gateway restart
```

Voer vervolgens uit:

```bash
openclaw onboard --auth-choice gmi-api-key
```

Niet-interactieve configuraties kunnen `--gmi-api-key <key>` doorgeven of het volgende instellen:

```bash
export GMI_API_KEY="<your-gmi-api-key>" # pragma: allowlist secret
```

## Wanneer kies je GMI?

- Je wilt een gehost OpenAI-compatibel eindpunt in plaats van een lokale modelserver.
- Je wilt meerdere commerciële modelfamilies en modelfamilies met open gewichten uitproberen via één
  provideraccount.
- Je wilt een fallbackprovider met andere upstreamroutering dan DeepInfra,
  OpenRouter, Together of de rechtstreekse API's van leveranciers.
- Je hebt GMI-specifieke model-id's, prijzen of accountinstellingen nodig.

Kies in plaats daarvan de rechtstreekse provider van de leverancier wanneer je leveranciersspecifieke functies nodig hebt
die GMI niet via zijn OpenAI-compatibele route aanbiedt. Kies een lokale
provider zoals LM Studio, Ollama, SGLang of vLLM wanneer gegevenslokaliteit of lokale
GPU-controle belangrijker is dan het gemak van hosting.

## Modellen

De Plugin-catalogus bevat als uitgangspunt veelgebruikte GMI Cloud-route-id's:

| Modelverwijzing                    | Invoer       | Context   | Maximale uitvoer |
| ---------------------------------- | ------------ | --------- | ---------- |
| `gmi/anthropic/claude-sonnet-4.6`  | tekst + afbeelding | 200,000   | 64,000     |
| `gmi/deepseek-ai/DeepSeek-V3.2`    | tekst         | 163,840   | 65,536     |
| `gmi/google/gemini-3.1-flash-lite` | tekst + afbeelding | 1,048,576 | 65,536     |
| `gmi/moonshotai/Kimi-K2.5`         | tekst + afbeelding | 262,144   | 65,536     |
| `gmi/openai/gpt-5.4`               | tekst + afbeelding | 400,000   | 128,000    |
| `gmi/zai-org/GLM-5.1-FP8`          | tekst         | 202,752   | 65,536     |

De catalogus is een uitgangspunt en geen garantie dat elk account elk model op
elk moment kan aanroepen. Geef weer wat de geconfigureerde provider in jouw omgeving meldt:

```bash
openclaw models list --provider gmi
```

## Problemen oplossen

- `401` of `403`: controleer of `GMI_API_KEY` is ingesteld voor het proces dat
  OpenClaw uitvoert, of voer onboarding opnieuw uit om de sleutel in het authenticatieprofiel van de provider op te slaan.
- Fouten voor onbekende modellen: controleer of het model in je GMI-account bestaat en gebruik de
  volledige `gmi/<route-id>`-verwijzing die door `openclaw models list --provider gmi` wordt weergegeven.
- Incidentele providerfouten: probeer een andere GMI-route of configureer GMI als
  fallback in plaats van als enige primaire modelprovider.

## Gerelateerd

- [Modelproviders](/nl/concepts/model-providers)
- [Alle providers](/nl/providers/index)
