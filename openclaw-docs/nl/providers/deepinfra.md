---
read_when:
    - Je wilt één API-sleutel voor de beste open-source-LLM's
    - Je wilt modellen uitvoeren via de API van DeepInfra in OpenClaw
summary: Gebruik de uniforme API van DeepInfra om toegang te krijgen tot de populairste opensource- en frontiermodellen in OpenClaw
title: DeepInfra
x-i18n:
    generated_at: "2026-07-27T05:46:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a63bdd4ffd2189cde50f0ee601fd7ee32ca86c943a9899072f0c140823608004
    source_path: providers/deepinfra.md
    workflow: 16
---

DeepInfra routeert verzoeken naar populaire opensource- en frontiermodellen via één OpenAI-compatibel eindpunt en één API-sleutel. De meeste OpenAI-SDK's werken ermee door de basis-URL te wijzigen.

## Plugin installeren

```bash
openclaw plugins install @openclaw/deepinfra-provider
openclaw gateway restart
```

## Een API-sleutel verkrijgen

1. Meld je aan op [deepinfra.com](https://deepinfra.com/)
2. Ga naar Dashboard / Keys en genereer een sleutel, of gebruik de automatisch aangemaakte sleutel

## Instellen via de CLI

```bash
openclaw onboard --deepinfra-api-key <key>
```

Of stel de omgevingsvariabele in:

```bash
export DEEPINFRA_API_KEY="<your-deepinfra-api-key>" # pragma: allowlist secret
```

## Configuratiefragment

```json5
{
  env: { DEEPINFRA_API_KEY: "<your-deepinfra-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "deepinfra/deepseek-ai/DeepSeek-V4-Flash" },
    },
  },
}
```

## Ondersteunde oppervlakken

Chat, afbeeldingsgeneratie en videogeneratie vernieuwen hun modelcatalogi live vanuit `https://api.deepinfra.com/v1/openai/models?sort_by=openclaw&filter=with_meta`
zodra `DEEPINFRA_API_KEY` is geconfigureerd. Live detectie breidt de lijst met
selecteerbare modellen uit; het standaardmodel per oppervlak blijft de statische waarde
hieronder. Andere oppervlakken gebruiken statische catalogi totdat ze naar dezelfde
livecatalogus worden overgezet.

| Oppervlak                | Standaardmodel                                                                 | OpenClaw-configuratie/-tool                           |
| ------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| Chat/modelprovider       | `deepseek-ai/DeepSeek-V4-Flash` (livecatalogus voegt meer chatmodellen toe)                 | `agents.defaults.model`                                    |
| Afbeeldingen genereren/bewerken | `black-forest-labs/FLUX-1-schnell` (livecatalogus voegt meer `image-gen`-modellen toe) | `image_generate`, `agents.defaults.mediaModels.image` |
| Mediabegrip              | `moonshotai/Kimi-K2.5` voor afbeeldingen                                           | begrip van inkomende afbeeldingen                     |
| Spraak-naar-tekst        | `openai/whisper-large-v3-turbo`                                                             | transcriptie van inkomende audio                      |
| Tekst-naar-spraak        | `hexgrad/Kokoro-82M`                                                             | `tts.provider: "deepinfra"`                                    |
| Videogeneratie           | `Pixverse/Pixverse-T2V` (livecatalogus voegt meer `video-gen`-modellen toe)  | `video_generate`, `agents.defaults.mediaModels.video` |
| Geheugen-embeddings      | `BAAI/bge-m3`                                                             | `memory.search.provider: "deepinfra"`                                    |

DeepInfra biedt ook herrangschikking, classificatie, objectdetectie en andere
native modeltypen. OpenClaw heeft nog geen providercontract voor die categorieën,
dus deze Plugin registreert ze niet.

## Beschikbare modellen

OpenClaw detecteert DeepInfra-modellen dynamisch zodra een sleutel is geconfigureerd. Gebruik
`/models deepinfra` of `openclaw models list --provider deepinfra` om de
huidige lijst te bekijken.

Elk model op [deepinfra.com](https://deepinfra.com/) werkt met het
voorvoegsel `deepinfra/`:

```text
deepinfra/deepseek-ai/DeepSeek-V4-Flash
deepinfra/deepseek-ai/DeepSeek-V3.2
deepinfra/MiniMaxAI/MiniMax-M2.5
deepinfra/moonshotai/Kimi-K2.5
deepinfra/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B
deepinfra/zai-org/GLM-5.1
...en nog veel meer
```

## Opmerkingen

- Modelreferenties zijn `deepinfra/<provider>/<model>` (bijvoorbeeld `deepinfra/Qwen/Qwen3-Max`).
- Standaard chatmodel: `deepinfra/deepseek-ai/DeepSeek-V4-Flash`
- Basis-URL: `https://api.deepinfra.com/v1/openai`
- Videogeneratie gebruikt het OpenAI-compatibele asynchrone eindpunt `https://api.deepinfra.com/v1/openai/videos` (indienen en vervolgens pollen). Een geconfigureerde `baseUrl` wordt gerespecteerd. `openclaw doctor --fix` migreert verouderde waarden van `nativeBaseUrl` of `/v1/inference` op `api.deepinfra.com` automatisch naar `baseUrl`; aangepaste native eindpunten zijn uitgefaseerd met een doctormelding en vereisen een handmatig geconfigureerde OpenAI-compatibele `baseUrl`. Videogeneratie mislukt met een foutmelding die aangeeft welke actie nodig is (voordat een verzoek wordt verzonden) zolang `baseUrl` nog naar het uitgefaseerde oppervlak `/v1/inference` verwijst.

## Gerelateerd

- [Modelproviders](/nl/concepts/model-providers)
- [Alle providers](/nl/providers/index)
