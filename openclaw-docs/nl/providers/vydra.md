---
read_when:
    - Je wilt Vydra-mediageneratie in OpenClaw
    - Je hebt uitleg nodig voor het instellen van een Vydra-API-sleutel
summary: Gebruik Vydra voor afbeeldingen, video en spraak in OpenClaw
title: Vydra
x-i18n:
    generated_at: "2026-07-27T05:31:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cc3856c2dd740e87d70d7eedefd9eae7905ab547aa0d68a1c479a305c59b2982
    source_path: providers/vydra.md
    workflow: 16
---

De meegeleverde Vydra-plugin voegt het volgende toe:

- Afbeeldingen genereren via `vydra/grok-imagine`
- Video's genereren via `vydra/veo3` (tekst-naar-video) en `vydra/kling` (afbeelding-naar-video)
- Spraaksynthese via Vydra's door ElevenLabs ondersteunde TTS-route

OpenClaw gebruikt dezelfde `VYDRA_API_KEY` voor alle drie de mogelijkheden.

| Eigenschap      | Waarde                                                                    |
| --------------- | ------------------------------------------------------------------------- |
| Provider-id     | `vydra`                                                        |
| Plugin          | meegeleverd, `enabledByDefault: true`                                          |
| Auth-omgevingsvariabele | `VYDRA_API_KEY`                                                |
| Onboardingvlag  | `--auth-choice vydra-api-key`                                                        |
| Directe CLI-vlag | `--vydra-api-key <key>`                                                       |
| Contracten      | `imageGenerationProviders`, `videoGenerationProviders`, `speechProviders`               |
| Basis-URL       | `https://www.vydra.ai/api/v1` (gebruik de host `www`)                   |

<Warning>
Gebruik `https://www.vydra.ai/api/v1` als basis-URL. De apexhost van Vydra (`https://vydra.ai/api/v1`) verwijst momenteel door naar `www`. Sommige HTTP-clients laten `Authorization` weg bij die doorverwijzing tussen hosts, waardoor een geldige API-sleutel resulteert in een misleidende authenticatiefout. De meegeleverde Plugin normaliseert elke geconfigureerde `vydra.ai`-basis-URL naar `www.vydra.ai` om dit te voorkomen.
</Warning>

## Installatie

<Steps>
  <Step title="Interactieve onboarding uitvoeren">
    ```bash
    openclaw onboard --auth-choice vydra-api-key
    ```

    Of stel de omgevingsvariabele rechtstreeks in:

    ```bash
    export VYDRA_API_KEY="vydra_live_..."
    ```

  </Step>
  <Step title="Een standaardmogelijkheid kiezen">
    Kies een of meer van de onderstaande mogelijkheden (afbeelding, video of spraak) en pas de bijbehorende configuratie toe.
  </Step>
</Steps>

## Mogelijkheden

<AccordionGroup>
  <Accordion title="Afbeeldingen genereren">
    Standaard en enige meegeleverde afbeeldingsmodel:

    - `vydra/grok-imagine`

    Stel dit in als de standaardprovider voor afbeeldingen:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "vydra/grok-imagine",
          },
        },
      },
    }
    ```

    De meegeleverde ondersteuning is alleen voor tekst-naar-afbeelding, met maximaal één afbeelding per aanvraag. De gehoste bewerkingsroutes van Vydra verwachten externe afbeeldings-URL's en de meegeleverde Plugin voegt geen Vydra-specifieke uploadbridge toe.

    <Note>
    Zie [Afbeeldingen genereren](/nl/tools/image-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
    </Note>

  </Accordion>

  <Accordion title="Video's genereren">
    Geregistreerde videomodellen:

    - `vydra/veo3` voor tekst-naar-video (weigert invoer met afbeeldingsreferenties)
    - `vydra/kling` voor afbeelding-naar-video (vereist precies één externe afbeeldings-URL)

    Stel Vydra in als de standaardprovider voor video:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "vydra/veo3",
          },
        },
      },
    }
    ```

    Opmerkingen:

    - `vydra/kling` weigert uploads van lokale bestanden vooraf; alleen een verwijzing naar een externe afbeeldings-URL werkt.
    - De HTTP-route `kling` van Vydra is niet consistent geweest over de vraag of deze `image_url` of `video_url` vereist; de meegeleverde provider verzendt dezelfde externe afbeeldings-URL in beide velden.
    - De meegeleverde Plugin blijft terughoudend en stuurt geen ongedocumenteerde stijlopties door, zoals beeldverhouding, resolutie, watermerk of gegenereerde audio.

    <Note>
    Zie [Video's genereren](/nl/tools/video-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
    </Note>

  </Accordion>

  <Accordion title="Live tests voor video">
    Providerspecifieke live testdekking:

    ```bash
    OPENCLAW_LIVE_TEST=1 \
    OPENCLAW_LIVE_VYDRA_VIDEO=1 \
    pnpm test:live -- extensions/vydra/vydra.live.test.ts
    ```

    Het meegeleverde livebestand voor Vydra dekt:

    - `vydra/veo3` tekst-naar-video
    - `vydra/kling` afbeelding-naar-video met een externe afbeeldings-URL

    Overschrijf indien nodig de externe afbeeldingsfixture:

    ```bash
    export OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL="https://example.com/reference.png"
    ```

  </Accordion>

  <Accordion title="Spraaksynthese">
    Stel Vydra in als spraakprovider:

    ```json5
    {
      tts: {
        provider: "vydra",
        providers: {
          vydra: {
            apiKey: "${VYDRA_API_KEY}",
            voiceId: "21m00Tcm4TlvDq8ikWAM",
          },
        },
      },
    }
    ```

    Standaardwaarden:

    - Model: `elevenlabs/tts`
    - Stem-id: `21m00Tcm4TlvDq8ikWAM` ("Rachel")

    De meegeleverde Plugin stelt deze ene beproefde standaardstem beschikbaar en retourneert MP3-audiobestanden.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Provideroverzicht" href="/nl/providers/index" icon="list">
    Bekijk alle beschikbare providers.
  </Card>
  <Card title="Afbeeldingen genereren" href="/nl/tools/image-generation" icon="image">
    Gedeelde parameters voor de afbeeldingstool en providerselectie.
  </Card>
  <Card title="Video's genereren" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor de videotool en providerselectie.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults" icon="gear">
    Standaardwaarden voor agents en modelconfiguratie.
  </Card>
</CardGroup>
