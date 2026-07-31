---
read_when:
    - Je wilt Runway-videogeneratie gebruiken in OpenClaw
    - Je hebt de Runway-API-sleutel/-omgevingsconfiguratie nodig
    - Je wilt Runway instellen als de standaardvideoprovider
summary: Instellen van Runway-videogeneratie in OpenClaw
title: Runway
x-i18n:
    generated_at: "2026-07-27T06:10:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a56e768893e327b56d70e8b8c2d426123a861b3cf05c0107d98104e2cee856c
    source_path: providers/runway.md
    workflow: 16
---

OpenClaw wordt geleverd met een gebundelde `runway`-provider voor gehoste videogeneratie, die standaard is ingeschakeld en is geregistreerd voor het `videoGenerationProviders`-contract.

| Eigenschap            | Waarde                                                            |
| --------------------- | ----------------------------------------------------------------- |
| Provider-id           | `runway`                                                |
| Plugin                | gebundeld, `enabledByDefault: true`                                     |
| Omgevingsvariabelen voor authenticatie | `RUNWAYML_API_SECRET` (canoniek) of `RUNWAY_API_KEY` |
| Onboarding-vlag       | `--auth-choice runway-api-key`                                                |
| Directe CLI-vlag      | `--runway-api-key <key>`                                                |
| API                   | Runway-taakgebaseerde videogeneratie (`GET /v1/tasks/{id}`-polling) |
| Standaardmodel        | `runway/gen4.5`                                                |

## Aan de slag

<Steps>
  <Step title="Stel de API-sleutel in">
    ```bash
    openclaw onboard --auth-choice runway-api-key
    ```
  </Step>
  <Step title="Stel Runway in als de standaardprovider voor video">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "runway/gen4.5"
    ```
  </Step>
  <Step title="Genereer een video">
    Vraag de agent om een video te genereren. Runway wordt automatisch gebruikt.
  </Step>
</Steps>

## Ondersteunde modi en modellen

De provider biedt zeven Runway-modellen, verdeeld over drie modi. Dezelfde model-id kan voor meer dan één modus dienen (zo werkt `gen4.5` voor zowel tekst-naar-video als afbeelding-naar-video).

| Modus                  | Modellen                                                               | Referentie-invoer                 |
| ---------------------- | ---------------------------------------------------------------------- | --------------------------------- |
| Tekst-naar-video       | `gen4.5` (standaard), `veo3.1`, `veo3.1_fast`, `veo3` | Geen                              |
| Afbeelding-naar-video  | `gen4.5`, `gen4_turbo`, `gen3a_turbo`, `veo3.1`, `veo3.1_fast`, `veo3` | 1 lokale of externe afbeelding   |
| Video-naar-video       | `gen4_aleph`                                                    | 1 lokale of externe video        |

Lokale afbeeldings- en videoverwijzingen worden ondersteund via data-URI's.

| Beeldverhoudingen             | Toegestane waarden                           |
| ----------------------------- | -------------------------------------------- |
| Tekst-naar-video              | `16:9`, `9:16`       |
| Afbeeldings- en videobewerkingen | `1:1`, `16:9`, `9:16`, `3:4`, `4:3`, `21:9` |

<Warning>
  Video-naar-video vereist momenteel `runway/gen4_aleph`. Andere Runway-model-id's weigeren videoverwijzingen als invoer.
</Warning>

<Note>
  Als je een Runway-model-id uit de verkeerde kolom kiest, treedt er een expliciete fout op voordat het API-verzoek OpenClaw verlaat. De provider valideert `model` aan de hand van de lijst met toegestane waarden voor de modus (`TEXT_ONLY_MODELS`, `IMAGE_MODELS`, `VIDEO_MODELS`) in `extensions/runway/video-generation-provider.ts`.
</Note>

## Configuratie

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Aliassen voor omgevingsvariabelen">
    OpenClaw herkent zowel `RUNWAYML_API_SECRET` (canoniek) als `RUNWAY_API_KEY`.
    Beide variabelen authenticeren de Runway-provider.
  </Accordion>

  <Accordion title="Taakpolling">
    Runway gebruikt een taakgebaseerde API. Nadat een genereringsverzoek is ingediend, pollt OpenClaw
    `GET /v1/tasks/{id}` totdat de video gereed is. Voor het
    pollinggedrag is geen aanvullende configuratie nodig.
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Videogeneratie" href="/nl/tools/video-generation" icon="video">
    Gedeelde toolparameters, providerselectie en asynchroon gedrag.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults" icon="gear">
    Standaardinstellingen voor agents, waaronder het model voor videogeneratie.
  </Card>
</CardGroup>
