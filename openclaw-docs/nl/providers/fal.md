---
read_when:
    - Je wilt fal-beeldgeneratie gebruiken in OpenClaw
    - Je hebt de `FAL_KEY`-authenticatiestroom nodig
    - Je wilt fal-standaardinstellingen voor image_generate, video_generate of music_generate
summary: fal-configuratie voor het genereren van afbeeldingen, video en muziek in OpenClaw
title: Fal
x-i18n:
    generated_at: "2026-07-27T05:13:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9bd868aaf6771f6fa38bb8e2a83133460d150e2a5aa9e5b888e221c07f29e0ad
    source_path: providers/fal.md
    workflow: 16
---

OpenClaw wordt geleverd met een ingebouwde `fal`-provider voor gehoste generatie van afbeeldingen, video's en muziek.

| Eigenschap | Waarde                                                                           |
| -------- | ------------------------------------------------------------------------------- |
| Provider | `fal`                                                                           |
| Authenticatie     | `FAL_KEY` (canoniek; `FAL_API_KEY` werkt ook als fallback)                   |
| API      | fal-modeleindpunten (`https://fal.run`; videotaken gebruiken `https://queue.fal.run`) |
| Basis-URL | Overschrijven met `models.providers.fal.baseUrl`                                    |

## Aan de slag

<Steps>
  <Step title="Stel de API-sleutel in">
    ```bash
    openclaw onboard --auth-choice fal-api-key
    ```

    Niet-interactieve configuraties kunnen `--fal-api-key <key>` doorgeven of `FAL_KEY` exporteren.
    Tijdens de onboarding wordt ook `fal/fal-ai/flux/dev` ingesteld als het standaard afbeeldingsmodel wanneer
    er geen model is geconfigureerd.

  </Step>
  <Step title="Stel een standaard afbeeldingsmodel in">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/fal-ai/flux/dev",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

## Afbeeldingen genereren

De ingebouwde `fal`-provider voor afbeeldingsgeneratie gebruikt standaard
`fal/fal-ai/flux/dev`.

| Mogelijkheid     | Waarde                                                              |
| -------------- | ------------------------------------------------------------------ |
| Maximaal aantal afbeeldingen     | 4 per aanvraag; Krea 2: 1 per aanvraag                               |
| Overschrijvingen van grootte | `1024x1024`, `1024x1536`, `1536x1024`, `1024x1792`, `1792x1024`    |
| Beeldverhouding   | Overal ondersteund, behalve bij afbeelding-naar-afbeelding van Flux                    |
| Resolutie     | `1K`, `2K`, `4K` (modelspecifieke limieten hieronder)                          |
| Uitvoerindeling  | `png` (standaard) of `jpeg`; Krea 2 weigert overschrijvingen van `outputFormat` |

Bewerkingsaanvragen (referentieafbeeldingen via de gedeelde parameters `image` / `images`)
worden doorgestuurd naar een modelspecifiek bewerkingseindpunt met modelspecifieke referentielimieten:

| Modelfamilie              | Modelreferentie na `fal/`                 | Bewerkingseindpunt     | Maximaal aantal referentieafbeeldingen |
| ------------------------- | -------------------------------------- | ----------------- | -------------------- |
| Flux en andere fal-modellen | `fal-ai/flux/dev` (standaard)            | `/image-to-image` | 1                    |
| GPT Image                 | `openai/gpt-image-*`                   | `/edit`           | 10                   |
| Grok Imagine              | `xai/grok-imagine-image`               | `/edit`           | 3                    |
| Nano Banana (verouderd)      | `fal-ai/nano-banana`                   | `/edit`           | 3                    |
| Nano Banana 2             | `fal-ai/nano-banana-*`                 | `/edit`           | 14                   |
| Nano Banana 2 Lite        | `google/nano-banana-2-lite`            | `/edit`           | 14                   |
| Krea 2                    | `krea/v2/{medium,large}/text-to-image` | geen (stijlreferenties) | 10 stijlreferenties  |

<Warning>
Aanvragen voor afbeelding-naar-afbeelding van Flux ondersteunen **geen** overschrijvingen van `aspectRatio`. Bewerkingsaanvragen voor GPT
Image en Nano Banana 2 gebruiken het `/edit`-eindpunt van fal en accepteren
hints voor de beeldverhouding. Nano Banana 2 accepteert ook extra systeemeigen brede/hoge verhoudingen,
zoals `4:1`, `1:4`, `8:1` en `1:8`; Krea 2 valideert zijn eigen kleinere
verzameling beeldverhoudingen. Grok Imagine heeft een eigen lijst met verhoudingen (waaronder `2:1`,
`20:9`, `19.5:9` en hun omgekeerde waarden) en accepteert alleen de resoluties `1K`/`2K`;
de verouderde Nano Banana en Nano Banana 2 Lite weigeren overschrijvingen van `resolution`.
</Warning>

Krea 2-modellen gebruiken het systeemeigen Krea-payloadschema van fal. OpenClaw verzendt
`aspect_ratio`, `creativity` en `image_style_references` in plaats van de
algemene `image_size`-payload/payload voor het bewerkingseindpunt die Flux gebruikt. De modelreferenties zijn:

- `fal/krea/v2/medium/text-to-image`
- `fal/krea/v2/large/text-to-image`

Gebruik Medium voor snellere expressieve illustraties, anime, schilderijen en artistieke
stijlen. Gebruik Large voor tragere fotorealistische beelden, ruwe texturen, filmkorrel en gedetailleerde
weergaven. Krea gebruikt standaard `fal.creativity: "medium"`; ondersteunde waarden zijn
`raw`, `low`, `medium` en `high`.

Krea 2 biedt in het aanvraagschema van fal de beeldverhouding aan, niet `image_size`. Geef de voorkeur aan
`aspectRatio`; OpenClaw koppelt `size` aan de dichtstbijzijnde ondersteunde Krea-beeldverhouding
en weigert `resolution` voor Krea in plaats van deze te negeren.

Gebruik `outputFormat: "png"` wanneer je PNG-uitvoer wilt van fal-modellen die
`output_format` aanbieden. fal declareert in OpenClaw geen expliciete instelling voor een transparante achtergrond,
dus `background: "transparent"` wordt voor fal-modellen gerapporteerd als een genegeerde
overschrijving.
Krea 2-eindpunten bieden via fal geen aanvraagveld `output_format` aan, dus
OpenClaw weigert overschrijvingen van `outputFormat` voor Krea-aanvragen.

Krea 2 Medium gebruiken:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/krea/v2/medium/text-to-image",
      },
    },
  },
}
```

## Video's genereren

De ingebouwde `fal`-provider voor videogeneratie gebruikt standaard
`fal/fal-ai/minimax/video-01-live`.

| Mogelijkheid | Waarde                                                              |
| ---------- | ------------------------------------------------------------------ |
| Modi      | Tekst-naar-video, verwijzing naar één afbeelding, Seedance-referentie-naar-video |
| Uitvoering    | Door een wachtrij ondersteunde stroom voor indienen/status/resultaat voor langdurige taken       |
| Time-out    | Standaard 20 minuten per taak; status wordt elke 5 seconden opgevraagd       |

<AccordionGroup>
  <Accordion title="Beschikbare videomodellen">
    **MiniMax (standaard):**

    - `fal/fal-ai/minimax/video-01-live`

    **HeyGen-videoagent:**

    - `fal/fal-ai/heygen/v2/video-agent`

    **Kling en Wan:**

    - `fal/fal-ai/kling-video/v2.1/master/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/image-to-video`

    **Seedance 2.0:**

    - `fal/bytedance/seedance-2.0/fast/text-to-video`
    - `fal/bytedance/seedance-2.0/fast/image-to-video`
    - `fal/bytedance/seedance-2.0/fast/reference-to-video`
    - `fal/bytedance/seedance-2.0/text-to-video`
    - `fal/bytedance/seedance-2.0/image-to-video`
    - `fal/bytedance/seedance-2.0/reference-to-video`

    MiniMax Live- en HeyGen-aanvragen verzenden alleen de prompt plus een optionele
    enkele referentieafbeelding; andere overschrijvingen worden niet doorgestuurd. Seedance-modellen
    accepteren `aspectRatio`, `size`, `resolution`, tijdsduren van 4-15 seconden en
    een audioschakelaar.

  </Accordion>

  <Accordion title="Configuratievoorbeeld voor Seedance 2.0">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Configuratievoorbeeld voor referentie-naar-video met Seedance 2.0">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/reference-to-video",
          },
        },
      },
    }
    ```

    Referentie-naar-video accepteert maximaal 9 afbeeldingen, 3 video's en 3 audioreferenties
    via de gedeelde parameters `video_generate` `images`, `videos` en `audioRefs`,
    met maximaal 12 referentiebestanden in totaal. Audioreferenties vereisen
    minstens één afbeeldings- of videoreferentie in dezelfde aanvraag.

  </Accordion>

  <Accordion title="Configuratievoorbeeld voor HeyGen-videoagent">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/fal-ai/heygen/v2/video-agent",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Muziek genereren

De ingebouwde `fal`-plugin registreert ook een provider voor muziekgeneratie voor de
gedeelde tool `music_generate`.

| Mogelijkheid    | Waarde                                                                                                                    |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Standaardmodel | `fal/fal-ai/minimax-music/v2.6`                                                                                          |
| Modellen        | `fal-ai/minimax-music/v2.6` (mp3), `fal-ai/ace-step/prompt-to-audio` (wav), `fal-ai/stable-audio-25/text-to-audio` (wav) |
| Maximale duur  | 240 seconden                                                                                                              |
| Uitvoering       | Synchrone aanvraag plus download van gegenereerde audio                                                                        |

fal als standaard muziekprovider gebruiken:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "fal/fal-ai/minimax-music/v2.6",
      },
    },
  },
}
```

`fal-ai/minimax-music/v2.6` ondersteunt expliciete songteksten en een instrumentale modus,
maar niet beide in dezelfde aanvraag. ACE-Step en Stable Audio zijn
prompt-naar-audio-eindpunten; selecteer ze met de overschrijving `model` wanneer je
die modelfamilies wilt gebruiken. ACE-Step weigert expliciete songteksten; Stable Audio weigert
zowel songteksten als de instrumentale modus.

<Tip>
De bovenstaande tabellen en accordeons behandelen de modelfamilies waarvoor de ingebouwde
fal-provider speciale verwerking toepast. Andere fal-id's voor afbeeldingseindpunten kunnen nog steeds worden geselecteerd als
afbeeldingsmodel; ze worden behandeld als Flux (algemene `image_size`-payload, één
referentieafbeelding via `/image-to-image`).
</Tip>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Afbeeldingen genereren" href="/nl/tools/image-generation" icon="image">
    Gedeelde parameters voor het afbeeldingshulpmiddel en providerselectie.
  </Card>
  <Card title="Video's genereren" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor het videohulpmiddel en providerselectie.
  </Card>
  <Card title="Muziek genereren" href="/nl/tools/music-generation" icon="music">
    Gedeelde parameters voor het muziekhulpmiddel en providerselectie.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults" icon="gear">
    Agentstandaarden, waaronder de selectie van afbeeldings-, video- en muziekmodellen.
  </Card>
</CardGroup>
