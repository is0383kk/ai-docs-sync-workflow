---
read_when:
    - Je wilt Google Gemini-modellen gebruiken met OpenClaw
    - Je hebt de API-sleutel of OAuth-authenticatiestroom nodig
summary: Google Gemini instellen (API-sleutel + OAuth, afbeeldingen genereren, mediabegrip, TTS, zoeken op het web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-07-27T06:08:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf8db70bcebd425238e5f02ca12bdbcd75fa1c03d285ea127d4e3863892b3aa
    source_path: providers/google.md
    workflow: 16
---

De Google-Plugin biedt toegang tot Gemini-modellen via Google AI Studio, plus beeldgeneratie, mediabegrip (beeld/audio/video), tekst-naar-spraak en zoeken op het web via Gemini Grounding.

- Provider: `google`
- Authenticatie: `GEMINI_API_KEY` of `GOOGLE_API_KEY`
- API: Google Gemini API
- Runtime-optie: `agentRuntime.id: "google-gemini-cli"` hergebruikt Gemini CLI OAuth en houdt modelverwijzingen canoniek als `google/*`.

## Aan de slag

Kies de gewenste authenticatiemethode en volg de configuratiestappen.

<Tabs>
  <Tab title="API-sleutel">
    **Het meest geschikt voor:** standaardtoegang tot de Gemini API via Google AI Studio.

    <Steps>
      <Step title="Een API-sleutel verkrijgen">
        Maak een gratis sleutel aan in [Google AI Studio](https://aistudio.google.com/apikey).
      </Step>
      <Step title="De onboarding uitvoeren">
        ```bash
        openclaw onboard --auth-choice gemini-api-key
        ```

        Of geef de sleutel rechtstreeks door:

        ```bash
        openclaw onboard --non-interactive \
          --mode local \
          --auth-choice gemini-api-key \
          --gemini-api-key "$GEMINI_API_KEY"
        ```
      </Step>
      <Step title="Een standaardmodel instellen">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "google/gemini-3.1-pro-preview" },
            },
          },
        }
        ```
      </Step>
      <Step title="Controleren of het model beschikbaar is">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    <Tip>
    `GEMINI_API_KEY` en `GOOGLE_API_KEY` worden beide geaccepteerd. Gebruik de variant die je al hebt geconfigureerd.
    </Tip>

    Met een geconfigureerde API-sleutel vernieuwt OpenClaw de catalogus met
    tekstmodellen van Google AI Studio via de Gemini `models.list`-API. Nieuw
    uitgebrachte varianten van Gemini 3 Pro, Flash en Flash-Lite verschijnen
    daardoor in `openclaw models list --provider google` zonder dat je op een nieuwe OpenClaw-versie
    hoeft te wachten. Als detectie niet beschikbaar is, behoudt OpenClaw de
    meegeleverde reservecatalogus.

  </Tab>

  <Tab title="Gemini CLI (OAuth)">
    **Het meest geschikt voor:** aanmelden met je Google-account via Gemini CLI OAuth in plaats van een afzonderlijke API-sleutel te gebruiken.

    <Warning>
    De provider `google-gemini-cli` is een onofficiële integratie. Sommige
    gebruikers melden accountbeperkingen wanneer ze OAuth op deze manier
    gebruiken. Gebruik dit op eigen risico.
    </Warning>

    <Steps>
      <Step title="De Gemini CLI installeren">
        De lokale opdracht `gemini` moet beschikbaar zijn in `PATH`.

        ```bash
        # Homebrew
        brew install gemini-cli

        # of npm
        npm install -g @google/gemini-cli
        ```

        OpenClaw ondersteunt zowel installaties via Homebrew als globale
        npm-installaties, waaronder gangbare Windows/npm-indelingen.
      </Step>
      <Step title="Aanmelden via OAuth">
        ```bash
        openclaw models auth login --provider google-gemini-cli --set-default
        ```
      </Step>
      <Step title="Controleren of het model beschikbaar is">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    - Standaardmodel: `google/gemini-3.1-pro-preview`
    - Runtime: `google-gemini-cli`
    - Alias: `gemini-cli`

    De Gemini API-model-id van Gemini 3.1 Pro is `gemini-3.1-pro-preview`. OpenClaw accepteert het kortere `google/gemini-3.1-pro` als handige alias en normaliseert deze vóór provideraanroepen.

    **Omgevingsvariabelen:**

    - `OPENCLAW_GEMINI_OAUTH_CLIENT_ID` / `GEMINI_CLI_OAUTH_CLIENT_ID`
    - `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET` / `GEMINI_CLI_OAUTH_CLIENT_SECRET`

    <Note>
    Als Gemini CLI OAuth-verzoeken na het aanmelden mislukken, stel je
    `GOOGLE_CLOUD_PROJECT` of `GOOGLE_CLOUD_PROJECT_ID` in op de Gateway-host en probeer je het opnieuw.
    </Note>

    <Note>
    Als het aanmelden mislukt voordat de browserstroom begint, controleer je of
    de lokale opdracht `gemini` is geïnstalleerd en in `PATH` staat.
    </Note>

    De automatische detectie tijdens de onboarding vermeldt een bestaande
    Gemini CLI-aanmelding, maar test deze nooit automatisch omdat Gemini CLI
    geen test zonder tools heeft. Kies Gemini CLI OAuth of een Gemini API-sleutel
    om door te gaan.

    Modelverwijzingen met `google-gemini-cli/*` zijn verouderde
    compatibiliteitsaliassen. Nieuwe configuraties moeten modelverwijzingen met
    `google/*` plus de runtime `google-gemini-cli` gebruiken wanneer lokale
    uitvoering via Gemini CLI gewenst is.

  </Tab>
</Tabs>

<Note>
`google/gemini-3-pro-preview` is op 2026-03-09 buiten gebruik gesteld; gebruik in plaats daarvan `google/gemini-3.1-pro-preview`. Als je de configuratie van de Gemini API-sleutel opnieuw uitvoert (`openclaw onboard --auth-choice gemini-api-key` of `openclaw models auth login --provider google`), wordt een verouderd geconfigureerd standaardmodel vervangen door het huidige model.
</Note>

## Mogelijkheden

| Mogelijkheid                     | Ondersteund                    |
| -------------------------------- | ----------------------------- |
| Chatvoltooiingen                 | Ja                            |
| Beeldgeneratie                   | Ja                            |
| Muziekgeneratie                  | Ja                            |
| Tekst-naar-spraak                | Ja                            |
| Realtime spraak                  | Ja (Google Live API)          |
| Beeldbegrip                      | Ja                            |
| Audiotranscriptie                | Ja                            |
| Videobegrip                      | Ja                            |
| Zoeken op het web (Grounding)    | Ja                            |
| Denken/redeneren                 | Ja (Gemini 2.5+ / Gemini 3+)  |
| Gemma 4-modellen                 | Ja                            |

## Zoeken op het web

De meegeleverde webzoekprovider `gemini` gebruikt Gemini Google Search Grounding.
Configureer een specifieke zoeksleutel onder `plugins.entries.google.config.webSearch`,
of laat deze `models.providers.google.apiKey` hergebruiken na `GEMINI_API_KEY`:

```json5
{
  plugins: {
    entries: {
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // optioneel als GEMINI_API_KEY of models.providers.google.apiKey is ingesteld
            baseUrl: "https://generativelanguage.googleapis.com/v1beta", // valt terug op models.providers.google.baseUrl
            model: "gemini-2.5-flash",
          },
        },
      },
    },
  },
}
```

De volgorde van prioriteit voor inloggegevens is eerst de specifieke
`webSearch.apiKey`, vervolgens `GEMINI_API_KEY` en daarna `models.providers.google.apiKey`.
`webSearch.baseUrl` is optioneel en bestaat voor operatorproxy's of compatibele
Gemini API-eindpunten; als deze wordt weggelaten, hergebruikt Gemini-webzoekfunctie
`models.providers.google.baseUrl`. Zie [Zoeken met Gemini](/nl/tools/gemini-search) voor het
providerspecifieke gedrag van de tool.

<Tip>
Gemini 3-modellen gebruiken `thinkingLevel` in plaats van `thinkingBudget`.
OpenClaw wijst de besturingselementen voor redeneren van Gemini 3, Gemini 3.1 en
de alias `gemini-*-latest` toe aan `thinkingLevel`, zodat standaarduitvoeringen
en uitvoeringen met lage latentie geen uitgeschakelde waarden voor
`thinkingBudget` verzenden.

`/think adaptive` behoudt de dynamische denksemantiek van Google in plaats van
een vast OpenClaw-niveau te kiezen. Gemini 3 en Gemini 3.1 laten een vaste
`thinkingLevel` weg, zodat Google het niveau kan kiezen; Gemini 2.5 verzendt
de dynamische sentinelwaarde `thinkingBudget: -1` van Google.

Gemma 4-modellen (bijvoorbeeld `gemma-4-26b-a4b-it`) ondersteunen de denkmodus.
OpenClaw herschrijft `thinkingBudget` naar een ondersteunde Google-waarde voor
`thinkingLevel` voor Gemma 4. Als denken wordt ingesteld op
`off`, blijft denken uitgeschakeld in plaats van dat dit wordt
toegewezen aan `MINIMAL`.

Gemini 2.5 Pro werkt alleen in de denkmodus en weigert een expliciete
`thinkingBudget: 0`; OpenClaw verwijdert die waarde uit verzoeken voor Gemini 2.5
Pro in plaats van deze te verzenden.
</Tip>

## Beeldgeneratie

De meegeleverde provider voor beeldgeneratie `google` gebruikt
standaard `google/gemini-3.1-flash-image`.

- Ondersteunt ook `google/gemini-3-pro-image`
- Genereren: maximaal 4 beelden per verzoek
- Bewerkingsmodus: ingeschakeld, maximaal 5 invoerbeelden
- Geometriebesturing: `size`, `aspectRatio` en `resolution`

Google als standaardprovider voor beelden gebruiken:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image",
      },
    },
  },
}
```

<Note>
Zie [Beeldgeneratie](/nl/tools/image-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
</Note>

## Videogeneratie

De meegeleverde Plugin `google` registreert ook videogeneratie via de
gedeelde tool `video_generate`.

- Standaardvideomodel: `google/veo-3.1-fast-generate-preview`
- Modi: tekst-naar-video, beeld-naar-video en stromen met één video als referentie
- Ondersteunt `aspectRatio` (`16:9`, `9:16`) en `resolution` (`720P`, `1080P`); audio-uitvoer wordt momenteel niet ondersteund door Veo
- Ondersteunde tijdsduren: **4, 6 of 8 seconden** (andere waarden worden afgerond naar de dichtstbijzijnde toegestane waarde)

Google als standaardprovider voor video gebruiken:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

<Note>
Zie [Videogeneratie](/nl/tools/video-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
</Note>

## Muziekgeneratie

De meegeleverde Plugin `google` registreert ook muziekgeneratie via de
gedeelde tool `music_generate`.

- Standaardmuziekmodel: `google/lyria-3-clip-preview`
- Ondersteunt ook `google/lyria-3-pro-preview`
- Promptbesturing: `lyrics` en `instrumental`
- Uitvoerformaat: standaard `mp3`, plus `wav` op `google/lyria-3-pro-preview`
- Referentie-invoer: maximaal 10 beelden
- Uitvoeringen met een sessie worden losgekoppeld via de gedeelde taak-/statusstroom, waaronder `action: "status"`

Google als standaardprovider voor muziek gebruiken:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

<Note>
Zie [Muziekgeneratie](/nl/tools/music-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
</Note>

## Tekst-naar-spraak

De meegeleverde spraakprovider `google` gebruikt het TTS-pad van de
Gemini API met `gemini-3.1-flash-tts-preview`.

- Standaardstem: `Kore`
- Authenticatie: `tts.providers.google.apiKey`, `models.providers.google.apiKey`, `GEMINI_API_KEY` of `GOOGLE_API_KEY`
- Uitvoer: WAV voor gewone TTS-bijlagen, Opus voor spraakberichtdoelen, PCM voor Talk/telefonie
- Uitvoer voor spraakberichten: Google PCM wordt verpakt als WAV en met `ffmpeg` getranscodeerd naar 48 kHz Opus

Het batchpad voor Gemini TTS van Google retourneert de gegenereerde audio in het
voltooide antwoord `generateContent`. Gebruik voor gesproken gesprekken met de
laagste latentie de provider voor realtime spraak van Google, die wordt
aangedreven door de Gemini Live API, in plaats van batch-TTS.

Google als standaard-TTS-provider gebruiken:

```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        audioProfile: "Spreek professioneel met een rustige toon.",
      },
    },
  },
}
```

Gemini API TTS gebruikt prompts in natuurlijke taal voor stijlbesturing. Stel
`audioProfile` in om vóór de gesproken tekst een herbruikbare stijlprompt toe
te voegen. Stel `speakerName` in wanneer je prompttekst verwijst naar een
spreker met een naam.

Gemini API TTS accepteert in de tekst ook expressieve audiotags tussen vierkante
haken, zoals `[whispers]` of `[laughs]`. Plaats tags in een
`[[tts:text]]...[[/tts:text]]`-blok om ze buiten het zichtbare chatantwoord te houden en
tegelijkertijd naar TTS te verzenden:

```text
Hier staat de gewone antwoordtekst.

[[tts:text]][whispers] Hier staat de gesproken versie.[[/tts:text]]
```

<Note>
Een API-sleutel van Google Cloud Console die beperkt is tot de Gemini API, is
geldig voor deze provider. Dit is niet het afzonderlijke API-pad van Cloud
Text-to-Speech.
</Note>

## Realtime spraak

De meegeleverde Plugin `google` registreert een provider voor realtime
spraak, die wordt aangedreven door de Gemini Live API, voor audioverbindingen
aan de backend, zoals Voice Call en Google Meet.

| Instelling            | Configuratiepad                                                     | Standaardwaarde                                                                        |
| --------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Model                 | `plugins.entries.voice-call.config.realtime.providers.google.model` | `gemini-3.1-flash-live-preview`                                                       |
| Stem                  | `...google.voice`                                                   | `Kore`                                                                                |
| Temperatuur           | `...google.temperature`                                             | (niet ingesteld)                                                                      |
| VAD-startgevoeligheid | `...google.startSensitivity`                                        | (niet ingesteld)                                                                      |
| VAD-eindgevoeligheid  | `...google.endSensitivity`                                          | (niet ingesteld)                                                                      |
| Stilteduur            | `...google.silenceDurationMs`                                       | (niet ingesteld)                                                                      |
| Activiteitsafhandeling | `...google.activityHandling`                                        | Google-standaardwaarde, `start-of-activity-interrupts`                                        |
| Beurtdekking          | `...google.turnCoverage`                                            | Google-standaardwaarde, `audio-activity-and-all-video`                                        |
| Automatische VAD uitschakelen | `...google.automaticActivityDetectionDisabled`                      | `false`                                                                               |
| Sessie hervatten      | `...google.sessionResumption`                                       | `true`                                                                                |
| Contextcompressie     | `...google.contextWindowCompression`                                | `true`                                                                                |
| API-sleutel           | `...google.apiKey`                                                  | Valt terug op `models.providers.google.apiKey`, `GEMINI_API_KEY` of `GOOGLE_API_KEY` |

Voorbeeld van een realtimeconfiguratie voor Voice Call:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          realtime: {
            enabled: true,
            provider: "google",
            providers: {
              google: {
                model: "gemini-3.1-flash-live-preview",
                speakerVoice: "Kore",
                activityHandling: "start-of-activity-interrupts",
                turnCoverage: "audio-activity-and-all-video",
              },
            },
          },
        },
      },
    },
  },
}
```

<Note>
Google Live API gebruikt bidirectionele audio en functieaanroepen via een WebSocket.
OpenClaw past audio van de telefonie-/Meet-bridge aan voor Gemini's PCM Live API-stream en
houdt toolaanroepen binnen het gedeelde realtime spraakcontract. Laat `temperature`
niet ingesteld, tenzij je de sampling wilt wijzigen; OpenClaw laat niet-positieve waarden weg,
omdat Google Live voor `temperature: 0` transcripties zonder audio kan retourneren.
Transcriptie via de Gemini API wordt ingeschakeld zonder `languageCodes`; de huidige Google
SDK weigert hints voor taalcodes op dit API-pad.
</Note>

<Note>
Gemini 3.1 Live accepteert gesprekstekst via realtime-invoer en gebruikt
opeenvolgende functieaanroepen. OpenClaw laat voor dit model de oudere `NON_BLOCKING`,
planning van functieresponsen en velden voor affectieve dialogen weg. Geef de voorkeur aan
`thinkingLevel`; geconfigureerde positieve waarden voor `thinkingBudget` worden toegewezen aan het
dichtstbijzijnde ondersteunde niveau, terwijl `-1` de standaardwaarde van Google behoudt. Zie de
[vergelijking van Gemini Live-mogelijkheden](https://ai.google.dev/gemini-api/docs/live-api/capabilities).
</Note>

<Note>
Control UI Talk ondersteunt Google Live-browsersessies met beperkte tokens voor eenmalig gebruik.
In Video Talk stuurt de browser begrensde JPEG-frames rechtstreeks naar
Google Live, met het maximum van de provider van één frame per seconde. De functie
`describe_view` meldt of die camerastream actief is.
Cameraframes gaan niet door de Gateway. Realtime spraakproviders die alleen via de backend werken,
kunnen ook via het generieke Gateway-relaytransport worden uitgevoerd, waarbij
providerreferenties op de Gateway blijven.
</Note>

Voer voor liveverificatie door beheerders
`OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` uit.
De smoketest omvat ook OpenAI-backend-/WebRTC-paden; het Google-gedeelte maakt dezelfde
beperkte Live API-tokenvorm aan die Control UI Talk gebruikt, opent het
WebSocket-eindpunt van de browser, verzendt de initiële installatiepayload plus een JPEG-frame en
verifieert een tekstrespons en een functierondreis voor `describe_view`.

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Direct hergebruik van Gemini-cache">
    Voor rechtstreekse Gemini API-uitvoeringen (`api: "google-generative-ai"`) geeft OpenClaw
    een geconfigureerde `cachedContent`-handle door aan Gemini-aanvragen.

    - Configureer parameters per model of globaal met
      `cachedContent` of de verouderde `cached_content`
    - Parameters uit een specifiekere scope (modelniveau boven globaal) krijgen altijd voorrang.
      Als beide sleutels binnen dezelfde scope zijn ingesteld, krijgt `cached_content` voorrang.
      Gebruik slechts één sleutel per scope om verrassingen te voorkomen.
    - Voorbeeldwaarde: `cachedContents/prebuilt-context`
    - Gemini-cachetreffergebruik wordt genormaliseerd naar OpenClaw `cacheRead` vanuit
      upstream `cachedContentTokenCount`

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "google/gemini-2.5-pro": {
              params: {
                cachedContent: "cachedContents/prebuilt-context",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Gebruiksopmerkingen voor Gemini CLI">
    Bij gebruik van de OAuth-provider `google-gemini-cli` gebruikt OpenClaw standaard
    uitvoer van Gemini CLI `stream-json` en normaliseert het gebruik vanuit de uiteindelijke
    `stats`-payload. Verouderde overschrijvingen voor `--output-format json` gebruiken nog steeds de
    JSON-parser.

    - Gestreamde antwoordtekst is afkomstig van assistentgebeurtenissen van `message`.
    - Voor verouderde JSON-uitvoer is de antwoordtekst afkomstig uit het CLI JSON-veld `response`.
    - Het gebruik valt terug op `stats` wanneer de CLI `usage` leeg laat.
    - `stats.cached` wordt genormaliseerd naar OpenClaw `cacheRead`.
    - Als `stats.input` ontbreekt, leidt OpenClaw invoertokens af uit
      `stats.input_tokens - stats.cached`.

  </Accordion>

  <Accordion title="Omgevings- en daemonconfiguratie">
    Als de Gateway als daemon (launchd/systemd) wordt uitgevoerd, zorg er dan voor dat `GEMINI_API_KEY`
    beschikbaar is voor dat proces (bijvoorbeeld in `~/.openclaw/.env` of via
    `env.shellEnv`).
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Afbeeldingen genereren" href="/nl/tools/image-generation" icon="image">
    Gedeelde parameters voor afbeeldingstools en providerselectie.
  </Card>
  <Card title="Video's genereren" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor videotools en providerselectie.
  </Card>
  <Card title="Muziek genereren" href="/nl/tools/music-generation" icon="music">
    Gedeelde parameters voor muziektools en providerselectie.
  </Card>
</CardGroup>
