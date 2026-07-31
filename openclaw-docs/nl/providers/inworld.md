---
read_when:
    - Je wilt Inworld-spraaksynthese voor uitgaande antwoorden
    - Je hebt PCM-telefonie- of OGG_OPUS-spraaknotitie-uitvoer van Inworld nodig
summary: Inworld-streamingtekst-naar-spraak voor OpenClaw-antwoorden
title: Inworld
x-i18n:
    generated_at: "2026-07-27T06:06:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09560f5beda3b40d9c67f9408d34446f28ecddb8235fc0725c4265c813302946
    source_path: providers/inworld.md
    workflow: 16
---

Inworld is een provider voor streaming tekst-naar-spraak (TTS). In OpenClaw synthetiseert deze uitgaande antwoordaudio (standaard MP3, OGG_OPUS voor spraakberichten) en ruwe PCM-audio voor telefoniekanalen zoals Voice Call.

OpenClaw verstuurt verzoeken naar het streaming TTS-eindpunt van Inworld, voegt de geretourneerde base64-audiofragmenten samen tot één buffer en geeft het resultaat door aan de standaardpijplijn voor antwoordaudio.

| Eigenschap     | Waarde                                                          |
| -------------- | --------------------------------------------------------------- |
| Provider-id    | `inworld`                                              |
| Plugin         | officieel extern pakket (`@openclaw/inworld-speech`)                    |
| Contract       | `speechProviders` (alleen TTS)                                 |
| Auth-omgevingsvariabele | `INWORLD_API_KEY` (HTTP Basic, Base64-dashboardreferentie) |
| Basis-URL      | `https://api.inworld.ai`                                              |
| Standaardstem  | `Sarah`                                              |
| Standaardmodel | `inworld-tts-1.5-max`                                              |
| Uitvoer        | MP3 (standaard), OGG_OPUS (spraakberichten), PCM 22050 Hz (telefonie) |
| Website        | [inworld.ai](https://inworld.ai)                                |
| Documentatie   | [docs.inworld.ai/tts/tts](https://docs.inworld.ai/tts/tts)      |

## Plugin installeren

```bash
openclaw plugins install @openclaw/inworld-speech
openclaw gateway restart
```

## Aan de slag

<Steps>
  <Step title="Stel je API-sleutel in">
    Kopieer de referentie uit je Inworld-dashboard (Workspace > API Keys) en stel deze in als omgevingsvariabele. De waarde wordt ongewijzigd verzonden als HTTP Basic-referentie, dus codeer deze niet opnieuw met Base64 en zet deze niet om in een bearer-token.

    ```bash
    INWORLD_API_KEY=<base64-credential-from-dashboard>
    ```

  </Step>
  <Step title="Selecteer Inworld in tts">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "inworld",
        providers: {
          inworld: {
            voiceId: "Sarah",
            modelId: "inworld-tts-1.5-max",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Verstuur een bericht">
    Verstuur een antwoord via een verbonden kanaal. OpenClaw synthetiseert de audio met Inworld en levert deze als MP3 (of OGG_OPUS wanneer het kanaal een spraakbericht verwacht).
  </Step>
</Steps>

## Configuratieopties

| Optie         | Pad                                 | Beschrijving                                                        |
| ------------- | ----------------------------------- | ------------------------------------------------------------------- |
| `apiKey`      | `tts.providers.inworld.apiKey`      | Base64-dashboardreferentie. Valt terug op `INWORLD_API_KEY`.       |
| `baseUrl`     | `tts.providers.inworld.baseUrl`     | Overschrijf de basis-URL van de Inworld-API (standaard `https://api.inworld.ai`). |
| `voiceId`     | `tts.providers.inworld.voiceId`     | Stem-id (standaard `Sarah`). Verouderde alias: `speakerVoiceId`. |
| `modelId`     | `tts.providers.inworld.modelId`     | TTS-model-id (standaard `inworld-tts-1.5-max`).                        |
| `temperature` | `tts.providers.inworld.temperature` | Samplingtemperatuur, `0` (exclusief) tot `2` (optioneel). |

## Opmerkingen

<AccordionGroup>
  <Accordion title="Authenticatie">
    Inworld gebruikt HTTP Basic-authenticatie met één Base64-gecodeerde referentietekenreeks. Kopieer deze ongewijzigd uit het Inworld-dashboard. De provider verzendt deze als `Authorization: Basic <apiKey>` zonder verdere codering, dus codeer deze niet zelf met Base64 en geef geen token in bearer-stijl door. Zie [opmerkingen over TTS-authenticatie](/nl/tools/tts#inworld-primary) voor dezelfde waarschuwing.
  </Accordion>
  <Accordion title="Modellen">
    Ondersteunde model-id's: `inworld-tts-1.5-max` (standaard), `inworld-tts-1.5-mini`, `inworld-tts-1-max`, `inworld-tts-1`.
  </Accordion>
  <Accordion title="Audio-uitvoer">
    Antwoorden gebruiken standaard MP3. Wanneer het kanaaldoel `voice-note` is, vraagt OpenClaw Inworld om `OGG_OPUS`, zodat de audio als een systeemeigen spraakballon wordt afgespeeld. Telefoniesynthese gebruikt ruwe `PCM` op 22050 Hz als invoer voor de telefoniebridge.
  </Accordion>
  <Accordion title="Aangepaste eindpunten">
    Overschrijf de API-host met `tts.providers.inworld.baseUrl`. Schuine strepen aan het einde worden verwijderd voordat verzoeken worden verzonden.
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Tekst-naar-spraak" href="/nl/tools/tts" icon="waveform-lines">
    Overzicht van TTS, providers en de configuratie van `tts`.
  </Card>
  <Card title="Configuratie" href="/nl/gateway/configuration" icon="gear">
    Volledige configuratiereferentie, inclusief instellingen voor `tts`.
  </Card>
  <Card title="Providers" href="/nl/providers" icon="grid">
    Alle ondersteunde OpenClaw-providers.
  </Card>
  <Card title="Probleemoplossing" href="/nl/help/troubleshooting" icon="wrench">
    Veelvoorkomende problemen en stappen voor foutopsporing.
  </Card>
</CardGroup>
