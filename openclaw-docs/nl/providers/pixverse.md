---
read_when:
    - Je wilt PixVerse-videogeneratie gebruiken in OpenClaw
    - Je hebt de PixVerse API-sleutel-/omgevingsconfiguratie nodig
    - Je wilt PixVerse instellen als de standaardvideoprovider
summary: Instellen van PixVerse-videogeneratie in OpenClaw
title: PixVerse
x-i18n:
    generated_at: "2026-07-27T05:31:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dba881e877e3da4677a40dff736cb46de114337a1e0338ef8220dcd8e616f46
    source_path: providers/pixverse.md
    workflow: 16
---

OpenClaw biedt `pixverse` als officiële externe plugin voor gehoste PixVerse-videogeneratie. De plugin registreert de provider `pixverse` volgens het contract `videoGenerationProviders`.

| Eigenschap         | Waarde                                                               |
| ------------------ | -------------------------------------------------------------------- |
| Provider-id        | `pixverse`                                                   |
| Pluginpakket       | `@openclaw/pixverse-provider`                                                   |
| Omgevingsvariabele voor authenticatie | `PIXVERSE_API_KEY`                                  |
| Onboarding-vlag    | `--auth-choice pixverse-api-key`                                                   |
| Directe CLI-vlag   | `--pixverse-api-key <key>`                                                   |
| API                | PixVerse Platform API v2 (indiening via `video_id` plus pollen van het resultaat) |
| Standaardmodel     | `pixverse/v6`                                                   |
| Standaard-API-regio | Internationaal                                                      |

## Aan de slag

<Steps>
  <Step title="Installeer de plugin">
    ```bash
    openclaw plugins install @openclaw/pixverse-provider
    openclaw gateway restart
    ```
  </Step>
  <Step title="Stel de API-sleutel in">
    ```bash
    openclaw onboard --auth-choice pixverse-api-key
    ```

    De wizard vraagt om het internationale of Chinese eindpunt (zie API-regio
    hieronder) voordat `region` en `baseUrl` naar de providerconfiguratie worden geschreven.
    Niet-interactieve uitvoeringen (sleutel uit `--pixverse-api-key` of `PIXVERSE_API_KEY`)
    gebruiken standaard Internationaal.

    Onboarding stelt `agents.defaults.mediaModels.video.primary` ook in op
    `pixverse/v6` wanneer er nog geen standaardvideomodel is geconfigureerd.

  </Step>
  <Step title="Schakel over naar een bestaande standaardprovider voor video (optioneel)">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "pixverse/v6"
    ```
  </Step>
  <Step title="Genereer een video">
    Vraag de agent een video te genereren. PixVerse wordt automatisch gebruikt.
  </Step>
</Steps>

## Ondersteunde modi en modellen

De provider maakt PixVerse-generatiemodellen beschikbaar via het gedeelde videohulpmiddel van OpenClaw.

| Modus            | Modellen             | Referentie-invoer       |
| ---------------- | -------------------- | ----------------------- |
| Tekst-naar-video | `v6` (standaard), `c1` | Geen                    |
| Afbeelding-naar-video | `v6` (standaard), `c1` | 1 lokale of externe afbeelding |

Lokale afbeeldingsreferenties worden vóór de afbeelding-naar-video-aanvraag naar PixVerse geüpload. URL's van externe afbeeldingen worden als `image_url` doorgegeven via het eindpunt voor het uploaden van afbeeldingen van PixVerse.

| Optie            | Ondersteunde waarden                                                                                                             |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Duur             | 1-15 seconden (standaard 5)                                                                                                     |
| Resolutie        | `360P`, `540P`, `720P`, `1080P` (standaard `540P`; aanvragen voor `480P` worden toegewezen aan `540P`) |
| Beeldverhouding  | `16:9` (standaard), `4:3`, `1:1`, `3:4`, `9:16`, `2:3`, `3:2`, `21:9`; alleen tekst-naar-video, afbeelding-naar-video volgt de bronafbeelding |
| Gegenereerde audio | `audio: true`                                                                                                             |

<Note>
Het genereren van PixVerse-afbeeldingssjablonen is nog niet beschikbaar via `image_generate`. Die API wordt aangestuurd door sjabloon-id's, terwijl het gedeelde contract voor afbeeldingsgeneratie van OpenClaw momenteel geen PixVerse-specifieke getypeerde verzameling opties heeft.
</Note>

## Provideropties

De videoprovider accepteert deze optionele providerspecifieke sleutels:

| Optie                                | Type   | Effect                                        |
| ------------------------------------ | ------ | --------------------------------------------- |
| `seed`                   | number | Deterministische seed, 0 tot 2147483647       |
| `negativePrompt` / `negative_prompt` | string | Negatieve prompt                          |
| `quality`                   | string | PixVerse-kwaliteit, zoals `720p`  |
| `motionMode` / `motion_mode` | string | Bewegingsmodus voor afbeelding-naar-video (standaard `normal`) |
| `cameraMovement` / `camera_movement` | string | Voorinstelling voor PixVerse-camerabeweging |
| `templateId` / `template_id` | number | Id van geactiveerd PixVerse-sjabloon      |

## Configuratie

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "pixverse/v6",
      },
    },
  },
}
```

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="API-regio">
    | Regiowaarde     | Basis-URL van PixVerse-API                    |
    | --------------- | --------------------------------------------- |
    | `international` | `https://app-api.pixverse.ai/openapi/v2`                         |
    | `cn` | `https://app-api.pixverseai.cn/openapi/v2`                         |

    Stel `models.providers.pixverse.region` handmatig in wanneer je sleutel bij een
    specifieke PixVerse-platformregio hoort, of voer
    `openclaw onboard --auth-choice pixverse-api-key` uit om er een te kiezen in de
    configuratiewizard:

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            region: "cn", // "international" of "cn"
            baseUrl: "https://app-api.pixverseai.cn/openapi/v2",
            models: [],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Aangepaste basis-URL">
    Stel `models.providers.pixverse.baseUrl` alleen in wanneer het verkeer via een vertrouwde compatibele proxy wordt geleid.
    `baseUrl` heeft voorrang op `region`.

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            baseUrl: "https://app-api.pixverse.ai/openapi/v2",
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Taak pollen">
    PixVerse retourneert een `video_id` van de generatieaanvraag. OpenClaw pollt
    `/openapi/v2/video/result/{video_id}` elke 5 seconden totdat de taak
    slaagt, mislukt of de time-out bereikt (standaard 5 minuten; overschrijf dit met
    `agents.defaults.mediaModels.video.timeoutMs`).
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Videogeneratie" href="/nl/tools/video-generation" icon="video">
    Gedeelde hulpmiddelparameters, providerselectie en asynchroon gedrag.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults" icon="gear">
    Standaardinstellingen voor agents, waaronder het model voor videogeneratie.
  </Card>
</CardGroup>
