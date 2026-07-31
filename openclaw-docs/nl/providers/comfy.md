---
read_when:
    - Je wilt lokale ComfyUI-workflows gebruiken met OpenClaw
    - Je wilt Comfy Cloud gebruiken met workflows voor afbeeldingen, video of muziek
    - Je hebt de configuratiesleutels van de meegeleverde comfy-plugin nodig
summary: ComfyUI-workflowconfiguratie voor het genereren van afbeeldingen, video's en muziek in OpenClaw
title: ComfyUI
x-i18n:
    generated_at: "2026-07-27T05:18:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74150d202a422de8e0f4b2b82d5d12bd42eb46991e8ef688832208e1a2ff7793
    source_path: providers/comfy.md
    workflow: 16
---

OpenClaw wordt geleverd met een gebundelde `comfy`-plugin voor workflowgestuurde ComfyUI-uitvoeringen. De
plugin is volledig workflowgestuurd: OpenClaw koppelt geen algemene `size`-,
`aspectRatio`-, `resolution`-, `durationSeconds`- of TTS-achtige bedieningselementen aan
jouw graaf.

| Eigenschap        | Details                                                                          |
| ----------------- | -------------------------------------------------------------------------------- |
| Provider          | `comfy`                                                               |
| Model             | `comfy/workflow`                                                               |
| Gedeelde tools    | `image_generate`, `video_generate`, `music_generate`                       |
| Authenticatie     | Geen voor lokale ComfyUI; `COMFY_API_KEY` of `COMFY_CLOUD_API_KEY` voor Comfy Cloud |
| API               | ComfyUI `/prompt` / `/history` / `/view`; Comfy Cloud `/api/*` |

## Wat wordt ondersteund

- Afbeeldingen genereren en bewerken vanuit een workflow-JSON (voor bewerken is 1 geüploade referentieafbeelding nodig)
- Video's genereren vanuit een workflow-JSON, van tekst naar video of van afbeelding naar video (1 referentieafbeelding)
- Muziek/audio genereren via de gedeelde tool `music_generate`, met optioneel 1 referentieafbeelding
- Uitvoer downloaden van een geconfigureerd knooppunt, of van alle overeenkomende uitvoerknooppunten wanneer er geen is geconfigureerd

## Aan de slag

Kies tussen het uitvoeren van ComfyUI op je eigen computer en het gebruik van Comfy Cloud.

<Tabs>
  <Tab title="Lokaal">
    **Meest geschikt voor:** het uitvoeren van je eigen ComfyUI-instantie op je computer of LAN.

    <Steps>
      <Step title="Start ComfyUI lokaal">
        Zorg ervoor dat je lokale ComfyUI-instantie actief is (standaard `http://127.0.0.1:8188`).
      </Step>
      <Step title="Bereid je workflow-JSON voor">
        Exporteer of maak een ComfyUI-workflow-JSON-bestand. Noteer de knooppunt-ID's van het invoerknooppunt voor de prompt en het uitvoerknooppunt waarvan OpenClaw moet lezen.
      </Step>
      <Step title="Configureer de provider">
        Stel `mode: "local"` in en verwijs naar je workflowbestand. Minimaal voorbeeld voor afbeeldingen:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "local",
                  baseUrl: "http://127.0.0.1:8188",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Stel het standaardmodel in">
        Laat OpenClaw voor de geconfigureerde mogelijkheid naar het model `comfy/workflow` verwijzen:

        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Verifieer">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Comfy Cloud">
    **Meest geschikt voor:** workflows uitvoeren in Comfy Cloud zonder lokale GPU-resources te beheren.

    <Steps>
      <Step title="Verkrijg een API-sleutel">
        Meld je aan bij [comfy.org](https://comfy.org) en genereer een API-sleutel via het dashboard van je account.
      </Step>
      <Step title="Stel de API-sleutel in">
        Geef je sleutel op via een van deze methoden:

        ```bash
        # Onboarding-vlag
        openclaw onboard --comfy-api-key "your-key"

        # Omgevingsvariabele (aanbevolen voor daemons)
        export COMFY_API_KEY="your-key"

        # Alternatieve omgevingsvariabele
        export COMFY_CLOUD_API_KEY="your-key"

        # Of rechtstreeks in de configuratie
        openclaw config set plugins.entries.comfy.config.apiKey "your-key"
        ```
      </Step>
      <Step title="Bereid je workflow-JSON voor">
        Exporteer of maak een ComfyUI-workflow-JSON-bestand. Noteer de knooppunt-ID's van het invoerknooppunt voor de prompt en het uitvoerknooppunt.
      </Step>
      <Step title="Configureer de provider">
        Stel `mode: "cloud"` in en verwijs naar je workflowbestand:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "cloud",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```

        <Tip>
        In cloudmodus wordt `baseUrl` standaard ingesteld op `https://cloud.comfy.org`. Stel `baseUrl` alleen in voor een aangepast cloudeindpunt.
        </Tip>
      </Step>
      <Step title="Stel het standaardmodel in">
        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Verifieer">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Configuratie

Comfy ondersteunt gedeelde verbindingsinstellingen op het hoogste niveau en workflowsecties per mogelijkheid (`image`, `video`, `music`):

```json5
{
  plugins: {
    entries: {
      comfy: {
        config: {
          mode: "local",
          baseUrl: "http://127.0.0.1:8188",
          image: {
            workflowPath: "./workflows/flux-api.json",
            promptNodeId: "6",
            outputNodeId: "9",
          },
          video: {
            workflowPath: "./workflows/video-api.json",
            promptNodeId: "12",
            outputNodeId: "21",
          },
          music: {
            workflowPath: "./workflows/music-api.json",
            promptNodeId: "3",
            outputNodeId: "18",
          },
        },
      },
    },
  },
}
```

### Gedeelde sleutels

| Sleutel                | Type                   | Beschrijving                                                                          |
| ---------------------- | ---------------------- | ------------------------------------------------------------------------------------- |
| `mode`     | `"local"` of `"cloud"` | Verbindingsmodus. Standaard `"local"`.                           |
| `baseUrl`     | tekenreeks             | Standaard `http://127.0.0.1:8188` voor lokaal of `https://cloud.comfy.org` voor de cloud. |
| `apiKey`     | tekenreeks             | Optionele inline sleutel, als alternatief voor de omgevingsvariabelen `COMFY_API_KEY` / `COMFY_CLOUD_API_KEY`. |
| `allowPrivateNetwork`     | booleaans              | Sta een privé-/LAN-`baseUrl` toe in cloudmodus of een lokale privé-DNS-FQDN. |

<Note>
In de modus `local` werken letterlijke loopback-/privé-IP-adressen en servicenames met één label, zoals `http://comfyui:8188`, zonder `allowPrivateNetwork`. Privé-DNS-FQDN's die er openbaar uitzien, zoals `https://comfy.local.example.com`, vereisen `allowPrivateNetwork: true`. Vertrouwen in een privé-oorsprong blijft beperkt tot het geconfigureerde schema, de hostnaam en de poort; lokale omleidingen kunnen de geconfigureerde hostnaam niet verlaten, terwijl cloudomleidingen naar openbare CDN's worden gecontroleerd met het standaard-SSRF-beleid.
</Note>

### Sleutels per mogelijkheid

Deze sleutels zijn van toepassing binnen de secties `image`, `video` of `music`:

| Sleutel                      | Vereist | Standaard | Beschrijving                                                               |
| ---------------------------- | ------- | --------- | -------------------------------------------------------------------------- |
| `workflow` of `workflowPath` | Ja | -- | Inline workflow-JSON of het pad naar het ComfyUI-workflow-JSON-bestand. |
| `promptNodeId`           | Ja      | --        | Knooppunt-ID dat de tekstprompt ontvangt.                                  |
| `promptInputName`           | Nee     | `"text"` | Invoernaam op het promptknooppunt.                              |
| `outputNodeId`           | Nee     | --        | Knooppunt-ID waarvan de uitvoer wordt gelezen. Indien weggelaten, worden alle overeenkomende uitvoerknooppunten gebruikt. |
| `pollIntervalMs`           | Nee     | `1500` | Pollinginterval in milliseconden voor het voltooien van de taak. |
| `timeoutMs`           | Nee     | `300000` | Time-out in milliseconden voor de uitvoering van de workflow.   |

De secties `image` en `video` ondersteunen ook een invoerknooppunt voor referentieafbeeldingen:

| Sleutel                | Vereist                                      | Standaard | Beschrijving                                             |
| ---------------------- | -------------------------------------------- | --------- | -------------------------------------------------------- |
| `inputImageNodeId`     | Ja (wanneer een referentieafbeelding wordt doorgegeven) | -- | Knooppunt-ID dat de geüploade referentieafbeelding ontvangt. |
| `inputImageInputName`     | Nee                                          | `"image"` | Invoernaam op het afbeeldingsknooppunt.          |

`apiKey` accepteert een letterlijke tekenreeks of een [geheimverwijzing](/nl/gateway/configuration-reference#secrets)-object.

## Workflowdetails

<AccordionGroup>
  <Accordion title="Afbeeldingsworkflows">
    Stel het standaardafbeeldingsmodel in op `comfy/workflow`:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    **Voorbeeld van bewerken met een referentieafbeelding:**

    Voeg `inputImageNodeId` toe aan je afbeeldingsconfiguratie om afbeeldingen te kunnen bewerken met een geüploade referentieafbeelding:

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              image: {
                workflowPath: "./workflows/edit-api.json",
                promptNodeId: "6",
                inputImageNodeId: "7",
                inputImageInputName: "image",
                outputNodeId: "9",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Videoworkflows">
    Stel het standaardvideomodel in op `comfy/workflow`:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    Comfy-videoworkflows ondersteunen tekst-naar-video en afbeelding-naar-video via de geconfigureerde graaf.

    <Note>
    OpenClaw geeft geen invoervideo's door aan Comfy-workflows. Alleen tekstprompts en afzonderlijke referentieafbeeldingen worden als invoer ondersteund.
    </Note>

  </Accordion>

  <Accordion title="Muziekworkflows">
    De gebundelde plugin registreert een provider voor het genereren van muziek voor door workflows gedefinieerde audio- of muziekuitvoer, beschikbaar via de gedeelde tool `music_generate`. Deze accepteert een optionele referentieafbeelding (maximaal 1):

    ```text
    /tool music_generate prompt="Warme ambient-synthloop met zachte bandtextuur"
    ```

    Gebruik de configuratiesectie `music` om naar de JSON van je audioworkflow en het uitvoerknooppunt te verwijzen.

  </Accordion>

  <Accordion title="Achterwaartse compatibiliteit">
    Bestaande afbeeldingsconfiguratie op het hoogste niveau (zonder de geneste sectie `image`) blijft werken:

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              workflowPath: "./workflows/flux-api.json",
              promptNodeId: "6",
              outputNodeId: "9",
            },
          },
        },
      },
    }
    ```

    OpenClaw behandelt die verouderde vorm als de configuratie voor de afbeeldingsworkflow. Je hoeft niet onmiddellijk te migreren, maar de geneste secties `image` / `video` / `music` worden aanbevolen voor nieuwe configuraties. Als je alleen afbeeldingsgeneratie gebruikt, zijn de verouderde platte configuratie en de nieuwe geneste sectie `image` functioneel gelijkwaardig.

  </Accordion>

  <Accordion title="Live-tests">
    Voor de gebundelde plugin is optionele live-testdekking beschikbaar:

    ```bash
    OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
    ```

    De live-test slaat afzonderlijke gevallen voor afbeeldingen, video's of muziek over, tenzij de bijbehorende Comfy-workflowsectie is geconfigureerd.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Afbeeldingen genereren" href="/nl/tools/image-generation" icon="image">
    Configuratie en gebruik van de tool voor afbeeldingsgeneratie.
  </Card>
  <Card title="Video's genereren" href="/nl/tools/video-generation" icon="video">
    Configuratie en gebruik van de tool voor videogeneratie.
  </Card>
  <Card title="Muziek genereren" href="/nl/tools/music-generation" icon="music">
    Instellen van de tool voor muziek- en audiogeneratie.
  </Card>
  <Card title="Provideroverzicht" href="/nl/providers/index" icon="layers">
    Overzicht van alle providers en modelreferenties.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults" icon="gear">
    Volledige configuratiereferentie, inclusief standaardinstellingen voor agents.
  </Card>
</CardGroup>
