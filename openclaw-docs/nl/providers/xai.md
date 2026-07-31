---
read_when:
    - Je wilt Grok-modellen gebruiken in OpenClaw
    - Je configureert xAI-authenticatie of model-id's
summary: Gebruik xAI Grok-modellen in OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-07-27T05:59:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ae7b049649b08b6508b8331714fec3464628814629256ad23b584f0f8ca8b7
    source_path: providers/xai.md
    workflow: 16
---

OpenClaw wordt geleverd met een gebundelde `xai`-providerplugin voor Grok-modellen. Het
aanbevolen traject is Grok OAuth met een geschikt SuperGrok- of X Premium-
abonnement. Gateway, configuratie, routering en tools blijven lokaal; alleen Grok-
aanvragen gaan naar de API van xAI.

Voor OAuth zijn geen xAI API-sleutel en geen Grok Build-app vereist. xAI kan
Grok Build nog steeds op het toestemmingsscherm tonen omdat OpenClaw de gedeelde
OAuth-client van xAI gebruikt.

## Installatie

<Steps>
  <Step title="Nieuwe installatie">
    Voer de onboarding uit met installatie van de daemon en kies vervolgens xAI/Grok OAuth bij de
    model-/authenticatiestap:

    ```bash
    openclaw onboard --install-daemon
    ```

    Selecteer op een VPS of via SSH rechtstreeks xAI OAuth; dit gebruikt verificatie
    met een apparaatcode en heeft geen localhost-callback nodig:

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

  </Step>
  <Step title="Bestaande installatie">
    Meld je alleen aan bij xAI; voer niet de volledige onboarding opnieuw uit om alleen Grok te verbinden:

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    Stel Grok afzonderlijk in als standaardmodel:

    ```bash
    openclaw models set xai/grok-4.3
    ```

    Voer de volledige onboarding alleen opnieuw uit als je bewust keuzes voor Gateway,
    daemon, kanaal, werkruimte of andere installatieopties wilt wijzigen.

  </Step>
  <Step title="Traject met API-sleutel">
    Installatie met een API-sleutel blijft werken voor sleutels uit xAI Console en voor media-oppervlakken
    waarvoor providerconfiguratie op basis van een sleutel nodig is:

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

  </Step>
  <Step title="Kies een model">
    ```json5
    {
      agents: { defaults: { model: { primary: "xai/grok-4.3" } } },
    }
    ```
  </Step>
</Steps>

<Note>
OpenClaw gebruikt de xAI Responses API als het gebundelde xAI-transport. Dezelfde
referentie uit `openclaw models auth login --provider xai --method oauth` of
`--method api-key` stuurt ook `web_search` (provider-id `grok`), `x_search`,
`code_execution`, spraak/transcriptie en xAI-beeld-/videogeneratie aan. Als je
een xAI-sleutel opslaat onder `plugins.entries.xai.config.webSearch.apiKey`, gebruikt de
gebundelde xAI-modelprovider deze ook opnieuw als terugvaloptie.
</Note>

## Problemen met OAuth oplossen

- Gebruik voor SSH, Docker, VPS of andere externe installaties
  `openclaw models auth login --provider xai --method oauth`; dit gebruikt
  verificatie met een apparaatcode, niet een localhost-callback.
- Als het aanmelden slaagt maar Grok niet het standaardmodel is, voer je
  `openclaw models set xai/grok-4.3` uit.
- Controleer opgeslagen xAI-authenticatieprofielen:

  ```bash
  openclaw models auth list --provider xai
  openclaw models status
  ```

- xAI bepaalt welke accounts OAuth-API-tokens kunnen ontvangen. Als een account
  niet in aanmerking komt, gebruik je het traject met API-sleutel of controleer je het abonnement aan de kant van xAI.

<Tip>
Gebruik `xai-oauth` wanneer je je aanmeldt vanuit SSH, Docker of een VPS. OpenClaw toont een
URL en korte code; voltooi de aanmelding in een lokale browser terwijl het externe
proces xAI peilt naar de voltooide tokenuitwisseling.
</Tip>

## Ingebouwde catalogus

Selecteerbare id's in modelkiezers. De plugin kan oudere id's voor Grok 3,
Grok 4, Grok 4 Fast, Grok 4.1 Fast en Grok Code nog steeds verwerken voor bestaande configuraties;
zie [compatibiliteit met oudere versies en veranderende aliassen](#legacy-compatibility-and-moving-aliases).

| Familie        | Model-id's                                                    |
| -------------- | ------------------------------------------------------------ |
| Grok 4.5       | `grok-4.5` (aliassen: `grok-4.5-latest`, `grok-build-latest`) |
| Grok Build 0.1 | `grok-build-0.1`                                             |
| Grok 4.3       | `grok-4.3` (aliassen: `grok-4.3-latest`, `grok-latest`)       |
| Grok 4.20      | `grok-4.20-0309-reasoning`, `grok-4.20-0309-non-reasoning`   |

<Tip>
Gebruik `grok-4.5` voor algemene chats, programmeren en agentisch werk waar dit beschikbaar is.
Grok 4.3 blijft de regioveilige standaard voor de installatie; `grok-build-0.1` en beide
gedateerde Grok 4.20-varianten blijven selecteerbaar.
</Tip>

De context- en tokenkostenmetadata van de catalogus volgt de actuele
[modelpagina's](https://docs.x.ai/developers/models) en
[prijspagina](https://docs.x.ai/developers/pricing) van xAI. xAI hanteert hogere tarieven
wanneer een aanvraag de gedocumenteerde drempel voor lange context overschrijdt; de vaste
kostenvelden in de catalogus van OpenClaw registreren de tarieven voor korte context. Grok Build, de afzonderlijke
CLI voor programmeeragents van xAI, is beschikbaar op [x.ai/cli](https://x.ai/cli) en gebruikt momenteel
Grok 4.5.

## Functiedekking

De gebundelde plugin koppelt ondersteunde xAI-API's aan de gedeelde provider- en
toolcontracten van OpenClaw. Mogelijkheden die niet binnen het gedeelde contract passen, worden
hieronder of onder bekende beperkingen vermeld.

| xAI-mogelijkheid            | OpenClaw-oppervlak                       | Status                                               |
| -------------------------- | --------------------------------------- | ---------------------------------------------------- |
| Chat / Responses           | `xai/<model>`-modelprovider            | Ja                                                   |
| Webzoekopdracht aan serverzijde | `web_search`-provider `grok`            | Ja                                                   |
| X-zoekopdracht aan serverzijde | `x_search`-tool                         | Ja                                                   |
| Code-uitvoering aan serverzijde | `code_execution`-tool                   | Ja                                                   |
| Afbeeldingen               | `image_generate`                        | Ja                                                   |
| Video's                    | `video_generate`                        | Ja                                                   |
| Batchgewijze tekst-naar-spraak | `tts.provider: "xai"` / `tts`           | Ja                                                   |
| Streaming-TTS              | `textToSpeechStream`                    | Ja, via `wss://api.x.ai/v1/tts` (geen realtime spraak) |
| Batchgewijze spraak-naar-tekst | `tools.media.audio`-mediabegrip | Ja                                                   |
| Streaming-spraak-naar-tekst | Voice Call `streaming.provider: "xai"`  | Ja                                                   |
| Realtime spraak            | Talk `talk.realtime.provider: "xai"`    | Ja; Gateway-relay voor systeemeigen Talk-nodes       |
| Bestanden / batches        | Alleen compatibiliteit met generieke model-API | Geen volwaardige OpenClaw-tool                  |

<Note>
OpenClaw gebruikt de REST-API's van xAI voor afbeeldingen, video's, TTS en STT voor mediageneratie en
batchtranscriptie, de streaming-STT-WebSocket van xAI voor live transcriptie
van spraakoproepen, de Grok Voice Agent-WebSocket van xAI voor realtime Talk-sessies
en de Responses API voor chat-, zoek- en code-uitvoeringstools.
</Note>

### Compatibiliteit met oudere snelle modus

`/fast on` of `agents.defaults.models["xai/<model>"].params.fastMode: true`
herschrijft oudere xAI-configuraties nog steeds als volgt. Deze doel-id's worden
uitsluitend behouden voor compatibiliteit; gebruik actuele selecteerbare modellen voor nieuwe
configuraties.

| Bronmodel     | Doel voor snelle modus |
| ------------- | ------------------ |
| `grok-3`      | `grok-3-fast`      |
| `grok-3-mini` | `grok-3-mini-fast` |
| `grok-4`      | `grok-4-fast`      |
| `grok-4-0709` | `grok-4-fast`      |

### Compatibiliteit met oudere versies en veranderende aliassen

Oudere aliassen worden als volgt genormaliseerd:

| Oudere alias                                                  | Genormaliseerde id |
| ------------------------------------------------------------- | ---------------- |
| `grok-code-fast-1`, `grok-code-fast`, `grok-code-fast-1-0825` | `grok-build-0.1` |

De gedateerde 0309-id's zijn de selecteerbare catalogusvermeldingen. OpenClaw verzendt alle andere
actuele Grok 4.20-aliassen ongewijzigd, zodat xAI de zeggenschap behoudt over de semantiek van
stabiele, nieuwste, bèta-, experimentele en gedateerde aliassen. De algemene alias `grok-latest` wordt
ook ongewijzigd behouden.

xAI heeft de volgende exacte id's buiten gebruik gesteld. OpenClaw behoudt ze als verborgen compatibiliteits-
rijen voor uitgebrachte configuraties, met de limieten en prijzen van hun actuele
omleidingsdoelen:

| Buiten gebruik gestelde id's                                        | Huidig gedrag                    |
| -------------------------------------------------------------------- | -------------------------------- |
| `grok-4-1-fast-reasoning`, `grok-4-fast-reasoning`, `grok-4-0709`    | Grok 4.3 met `low`-redenering   |
| `grok-4-1-fast-non-reasoning`, `grok-4-fast-non-reasoning`, `grok-3` | Grok 4.3 met redenering uitgeschakeld |
| `grok-code-fast-1`                                                   | Grok Build 0.1                   |
| `grok-imagine-image-pro`                                             | Grok Imagine Image Quality       |

`openclaw doctor --fix` werkt opgeslagen standaardwaarden voor xAI-servertools en de
buiten gebruik gestelde slug voor kwaliteitsafbeeldingen bij, verwijdert verouderde gegenereerde catalogusrijen en herstelt
verouderde contextmetadata in actieve 4.20-rijen. Het zet actieve 4.20-
aliassen voor `beta-latest` niet vast op een gedateerde momentopname.

## Functies

<Warning>
  `x_search` en `code_execution` worden uitgevoerd op de servers van xAI. xAI brengt $5 per 1.000
  toolaanroepen in rekening, plus de invoer- en uitvoertokens van het model. Als de instelling
  `enabled` van een tool ontbreekt, stelt OpenClaw deze alleen beschikbaar voor een actief xAI-model.
  Voor een bekende niet-xAI-modelprovider is een expliciete `enabled: true` per tool vereist;
  een ontbrekende of niet-opgeloste provider wordt standaard geweigerd. xAI-authenticatie is altijd vereist
  en `enabled: false` schakelt de tool voor elke provider uit.
</Warning>

<AccordionGroup>
  <Accordion title="Zoeken op het web">
    De gebundelde `grok`-provider voor zoeken op het web geeft de voorkeur aan xAI OAuth en valt daarna terug
    op `XAI_API_KEY` of een webzoeksleutel van een plugin:

    ```bash
    openclaw models auth login --provider xai --method oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Accordion>

  <Accordion title="Videogeneratie">
    De gebundelde `xai`-plugin registreert videogeneratie via de gedeelde
    `video_generate`-tool.

    - Standaardmodel: `xai/grok-imagine-video`
    - Aanvullend model: `xai/grok-imagine-video-1.5`
    - Klassieke modi: tekst-naar-video, afbeelding-naar-video, generatie met referentieafbeeldingen,
      externe videobewerking en externe videoverlenging
    - Video 1.5-modus: alleen afbeelding-naar-video, met precies één afbeelding voor het eerste frame
    - Beeldverhoudingen: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3`;
      klassieke afbeelding-naar-video en afbeelding-naar-video met Video 1.5 nemen de verhouding van de bronafbeelding over wanneer
      deze is weggelaten
    - Resoluties: klassiek `480P`/`720P`; Video 1.5 ondersteunt ook `1080P`; alle
      generatiemodi gebruiken standaard `480P`
    - Duur: 1-15 seconden voor generatie/afbeelding-naar-video, 1-10 seconden bij
      gebruik van klassieke `reference_image`-rollen, 2-10 seconden voor klassieke verlenging
    - Generatie met referentieafbeeldingen: stel `imageRoles` in op `reference_image` voor
      elke aangeleverde afbeelding; xAI accepteert maximaal 7 van zulke afbeeldingen
    - Videobewerking/-verlenging neemt de beeldverhouding en resolutie van de invoervideo over;
      deze bewerkingen accepteren geen overschrijvingen van de geometrie
    - Standaardtime-out voor bewerkingen: 600 seconden, tenzij `video_generate.timeoutMs`
      of `agents.defaults.mediaModels.video.timeoutMs` is ingesteld

    <Warning>
    Lokale videobuffers worden niet geaccepteerd. Gebruik externe `http(s)`-URL's als invoer voor
    videobewerking/-verlenging. Afbeelding-naar-video accepteert lokale afbeeldingsbuffers omdat
    OpenClaw deze voor xAI codeert als data-URL's.
    </Warning>

    Video 1.5 herkent ook de identifiers `grok-imagine-video-1.5-preview` en
    `grok-imagine-video-1.5-2026-05-30` van xAI. OpenClaw stuurt de
    geselecteerde identifier ongewijzigd door, maar past dezelfde validatie voor uitsluitend afbeeldingen toe.

    xAI instellen als standaardvideoprovider:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "xai/grok-imagine-video",
          },
        },
      },
    }
    ```

    <Note>
    Zie [Videogeneratie](/nl/tools/video-generation) voor gedeelde toolparameters,
    providerselectie en failovergedrag.
    </Note>

  </Accordion>

  <Accordion title="Afbeeldingen genereren">
    De meegeleverde Plugin `xai` registreert het genereren van afbeeldingen via de gedeelde
    tool `image_generate`.

    - Standaard afbeeldingsmodel: `xai/grok-imagine-image`
    - Aanvullend model: `xai/grok-imagine-image-quality`
    - Modi: tekst-naar-afbeelding en bewerking van een referentieafbeelding
    - Referentie-invoer: één `image` of maximaal drie `images`
    - Beeldverhoudingen: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3`, `2:1`,
      `1:2`, `19.5:9`, `9:19.5`, `20:9`, `9:20`
    - Resoluties: `1K`, `2K`
    - Aantal: maximaal 4 afbeeldingen
    - Standaardtime-out voor bewerkingen: 600 seconden, tenzij `image_generate.timeoutMs`
      of `agents.defaults.mediaModels.image.timeoutMs` is ingesteld

    OpenClaw vraagt xAI om afbeeldingsreacties als `b64_json`, zodat gegenereerde media
    via het normale pad voor kanaalbijlagen kunnen worden opgeslagen en afgeleverd. Lokale
    referentieafbeeldingen worden omgezet in data-URL's; externe `http(s)`-referenties
    worden ongewijzigd doorgegeven.

    xAI als standaardprovider voor afbeeldingen gebruiken:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "xai/grok-imagine-image",
          },
        },
      },
    }
    ```

    <Note>
    xAI documenteert ook `quality`, `mask`, `user` en een beeldverhouding van `auto`.
    OpenClaw geeft momenteel alleen de gedeelde provideroverschrijdende afbeeldingsinstellingen door;
    deze uitsluitend native beschikbare opties worden niet via `image_generate` aangeboden.
    </Note>

  </Accordion>

  <Accordion title="Tekst-naar-spraak">
    De meegeleverde Plugin `xai` registreert tekst-naar-spraak via het gedeelde
    provideroppervlak `tts`.

    - Stemmen: geverifieerde livecatalogus van xAI; geef deze weer met
      `openclaw infer tts voices --provider xai`
    - Offline fallbackstemmen: `ara`, `eve`, `leo`, `rex`, `sal`
    - Standaardstem: `eve`
    - Aangepaste stem-ID's van accounts worden doorgegeven, ook als ze ontbreken in het
      antwoord van de ingebouwde catalogus
    - Indelingen: `mp3`, `wav`, `pcm`, `mulaw`, `alaw`
    - Taal: BCP-47-code of `auto`
    - Snelheid: providerspecifieke overschrijving van de snelheid
    - De native Opus-indeling voor spraakberichten wordt niet ondersteund

    xAI als standaard-TTS-provider gebruiken:

    ```json5
    {
      tts: {
        provider: "xai",
        providers: {
          xai: {
            voiceId: "eve",
          },
        },
      },
    }
    ```

    <Note>
    OpenClaw gebruikt xAI's batchendpoint `/v1/tts` voor gebufferde synthese,
    geverifieerde catalogusdetectie via `/v1/tts/voices` en native
    `wss://api.x.ai/v1/tts` voor streamingsynthese. Streaming is beperkt tot
    de native host `api.x.ai`, waardoor aangepaste waarden voor `baseUrl` op dit
    pad worden geweigerd. Het gebruikt de bestaande instellingen voor taal, stem, codec en snelheid; voor
    samplefrequentie en bitsnelheid gelden de standaardwaarden van xAI. Synthese van audiobestanden respecteert alle
    geconfigureerde codecs. Voor doelen voor spraakberichten wordt MP3 gebruikt voor streaming en gebufferde
    fallback, omdat de onbewerkte codecs van xAI geen metadata over codec of frequentie bevatten. De
    stream verzendt `text.delta` en vervolgens
    `text.done`, ontvangt `audio.delta`, `audio.done` of `error` en past een
    inactieve `timeoutMs` toe die bij elk audiofragment wordt vernieuwd. Dit staat los van
    realtime spraaksessies. Zie het contract van xAI's [Streaming-TTS-API](https://docs.x.ai/developers/rest-api-reference/inference/voice).
    </Note>

  </Accordion>

  <Accordion title="Spraak-naar-tekst">
    De meegeleverde Plugin `xai` registreert batchgewijze spraak-naar-tekst via het
    transcriptieoppervlak voor mediabegrip van OpenClaw.

    - Endpoint: xAI REST `/v1/stt`
    - Invoerpad: multipart-upload van audiobestanden
    - Modelselectie: xAI kiest het transcriptiemodel intern; het
      endpoint heeft geen modelkiezer
    - Wordt overal gebruikt waar transcriptie van inkomende audio `tools.media.audio` leest,
      waaronder segmenten van Discord-spraakkanalen en audiobijlagen van kanalen

    xAI afdwingen voor transcriptie van inkomende audio:

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "xai",
              },
            ],
          },
        },
      },
    }
    ```

    De taal kan via de gedeelde configuratie voor audiomedia of per
    transcriptieverzoek worden opgegeven. Promptaanwijzingen worden door het gedeelde OpenClaw-
    oppervlak geaccepteerd, maar de xAI REST STT-integratie geeft alleen het bestand en de taal door,
    omdat alleen die overeenkomen met het huidige openbare xAI-endpoint.

  </Accordion>

  <Accordion title="Streaming-spraak-naar-tekst">
    De meegeleverde Plugin `xai` registreert ook een realtime transcriptieprovider
    voor audio van live-spraakoproepen.

    - Endpoint: xAI WebSocket `wss://api.x.ai/v1/stt`
    - Standaardcodering: `mulaw`
    - Standaardsamplefrequentie: `8000`
    - Standaardeindpuntdetectie: `800ms`
    - Tussentijdse transcripties: standaard ingeschakeld

    De Twilio-mediastream van Voice Call verzendt G.711 mu-law-audioframes, zodat de
    xAI-provider deze frames rechtstreeks doorgeeft zonder transcodering:

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}",
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    Configuratie die eigendom is van de provider staat onder
    `plugins.entries.voice-call.config.streaming.providers.xai`. Ondersteunde
    sleutels zijn `apiKey`, `baseUrl`, `sampleRate`, `encoding` (`pcm`, `mulaw` of
    `alaw`), `interimResults`, `endpointingMs` en `language`.

    <Note>
    Deze streamingprovider is bedoeld voor het realtime transcriptiepad van Voice Call.
    Discord neemt korte segmenten op en gebruikt in plaats daarvan het batchgewijze
    transcriptiepad `tools.media.audio`.
    </Note>

  </Accordion>

  <Accordion title="Realtime spraak (Talk)">
    De meegeleverde Plugin `xai` registreert realtime Grok Voice Agent-sessies voor
    de Talk-modus via het gedeelde contract `registerRealtimeVoiceProvider`.

    - Endpoint: `wss://api.x.ai/v1/realtime?model=<voice-model>`
    - Standaardmodel: `grok-voice-latest`
    - Standaardstem: `eve`
    - Transport: `gateway-relay` (relaypaden voor iOS, Android en Control UI)
    - Audio: PCM16 24 kHz of G.711 µ-law 8 kHz
    - Onderbreken: xAI-server-VAD onderbreekt het antwoord; OpenClaw wist de afspeelwachtrij
      en kapt niet-afgespeelde providergeschiedenis af

    Talk configureren op de Gateway:

    ```json5
    {
      talk: {
        realtime: {
          provider: "xai",
          mode: "realtime",
          transport: "gateway-relay",
          brain: "agent-consult",
          providers: {
            xai: {
              model: "grok-voice-latest",
              voice: "eve",
              // Schakel dit alleen in als het opnieuw afspelen van sessies aan de providerzijde aanvaardbaar is.
              sessionResumption: false,
            },
          },
        },
      },
      env: { XAI_API_KEY: "xai-..." },
    }
    ```

    Configuratie die eigendom is van de provider wordt ook opgehaald uit
    `plugins.entries.voice-call.config.realtime.providers.xai` wanneer Voice Call
    of gedeelde realtime kiezers dezelfde providertoewijzing hergebruiken. Ondersteunde sleutels zijn
    `apiKey`, `baseUrl`, `model`, `voice`, `vadThreshold`, `silenceDurationMs`,
    `prefixPaddingMs`, `reasoningEffort` en `sessionResumption`.
    `reasoningEffort` accepteert alleen `high` of `none`, overeenkomstig de xAI Voice Agent-API.

    De server-VAD van xAI maakt altijd antwoorden aan en handelt audio-onderbrekingen af.
    Gebruik `consultRouting: "provider-direct"`; geforceerde transcriptroutering en het uitschakelen
    van onderbreking van invoeraudio worden niet ondersteund door het xAI Voice Agent-protocol.

    <Note>
    xAI OAuth of `XAI_API_KEY` kan realtime spraak verifiëren. WebRTC onder beheer van de
    browser maakt nog geen deel uit van dit provideroppervlak; gebruik Talk via gateway-relay op
    native Nodes of het relaypad van Control UI.
    </Note>

    <Note>
    `sessionResumption` is standaard `false`. Wanneer dit is ingesteld op `true`, vraagt OpenClaw
    xAI voldoende sessiestatus te bewaren om hetzelfde gesprek na een
    nieuwe verbinding te hervatten, waarna opnieuw verbinding wordt gemaakt met het geretourneerde gespreks-ID. Laat dit
    uitgeschakeld wanneer opnieuw afspelen of bewaren aan de providerzijde niet aanvaardbaar is; onderbroken
    sockets worden dan gesloten bij fouten in plaats van stilzwijgend een nieuw gesprek te starten.
    </Note>

  </Accordion>

  <Accordion title="Configuratie van x_search">
    De meegeleverde xAI-Plugin biedt `x_search` aan als OpenClaw-tool voor
    het doorzoeken van inhoud op X (voorheen Twitter) via Grok.

    Configuratiepad: `plugins.entries.xai.config.xSearch`

    | Sleutel           | Type    | Standaard                 | Beschrijving                                     |
    | ----------------- | ------- | ------------------------- | ------------------------------------------------ |
    | `enabled`         | boolean | Automatisch voor xAI-modellen | Uitschakelen of inschakelen voor een bekende niet-xAI-provider |
    | `model`           | string  | `grok-4.3`                | Model dat wordt gebruikt voor x_search-verzoeken |
    | `baseUrl`         | string  | -                         | Overschrijving van de basis-URL van xAI Responses |
    | `inlineCitations` | boolean | -                         | Inline bronvermeldingen opnemen in resultaten    |
    | `maxTurns`        | number  | -                         | Maximaal aantal gespreksbeurten                   |
    | `timeoutSeconds`  | number  | `30`                      | Time-out van verzoeken in seconden                |
    | `cacheTtlMinutes` | number  | `15`                      | Cachelevensduur in minuten                        |

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              xSearch: {
                enabled: true,
                model: "grok-4.3",
                baseUrl: "https://api.x.ai/v1",
                inlineCitations: true,
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Configuratie van code-uitvoering">
    De meegeleverde xAI-Plugin biedt `code_execution` aan als OpenClaw-tool voor
    externe code-uitvoering in de sandboxomgeving van xAI.

    Configuratiepad: `plugins.entries.xai.config.codeExecution`

    | Sleutel          | Type    | Standaard                | Beschrijving                                      |
    | ---------------- | ------- | ------------------------ | ------------------------------------------------ |
    | `enabled`        | boolean | Automatisch voor xAI-modellen | Uitschakelen of inschakelen voor een bekende niet-xAI-provider |
    | `model`          | string  | `grok-4.3`               | Model dat wordt gebruikt voor aanvragen voor code-uitvoering |
    | `maxTurns`       | number  | -                        | Maximaal aantal conversatiebeurten                |
    | `timeoutSeconds` | number  | `30`                     | Time-out van aanvragen in seconden                |

    <Note>
    Dit is uitvoering in een externe xAI-sandbox, niet lokale [`exec`](/nl/tools/exec).
    </Note>

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true,
                model: "grok-4.3",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Bekende beperkingen">
    - Voor xAI-authenticatie kan een API-sleutel, omgevingsvariabele, terugvalwaarde uit de pluginconfiguratie of OAuth met een geschikt xAI-account worden gebruikt. OAuth gebruikt verificatie via een apparaatcode zonder localhost-callback. xAI bepaalt welke accounts OAuth-API-tokens kunnen ontvangen en op de toestemmingspagina kan Grok Build worden weergegeven, hoewel OpenClaw de Grok Build-app niet vereist.
    - OpenClaw stelt de xAI-modelfamilie voor meerdere agents momenteel niet beschikbaar. xAI biedt deze modellen aan via de Responses API, maar ze accepteren niet de client-side of aangepaste tools die door de gedeelde agentlus van OpenClaw worden gebruikt. Zie de
      [beperkingen van xAI voor meerdere agents](https://docs.x.ai/developers/model-capabilities/text/multi-agent#limitations).
    - xAI Realtime-spraak biedt momenteel alleen het Talk-transport via een Gateway-relay. WebSocket-sessies van providers die door de browser worden beheerd, zijn nog niet gekoppeld in de Control UI.
    - xAI-afbeelding `quality`, afbeelding `mask` en aanvullende uitsluitend native beschikbare beeldverhoudingen worden pas beschikbaar gesteld wanneer de gedeelde tool `image_generate` overeenkomstige provideroverschrijdende bedieningselementen heeft.

  </Accordion>

  <Accordion title="Geavanceerde opmerkingen">
    - OpenClaw past automatisch xAI-specifieke compatibiliteitscorrecties voor toolschema's en toolaanroepen toe op het gedeelde runnerpad.
    - Native xAI-aanvragen gebruiken standaard `tool_stream: true`. Stel
      `agents.defaults.models["xai/<model>"].params.tool_stream` in op `false`
      om dit uit te schakelen.
    - De meegeleverde xAI-wrapper verwijdert niet-ondersteunde schemagrenzen voor aantallen van contains en niet-ondersteunde payloadsleutels voor *effort* bij redeneren voordat native xAI-aanvragen worden verzonden. Grok 4.5 ondersteunt een lage, gemiddelde en hoge inspanning (standaard hoog). Grok 4.3 ondersteunt geen, lage, gemiddelde en hoge inspanning (standaard laag). Andere xAI-modellen die kunnen redeneren, bieden geen configureerbare regeling voor de inspanning, maar vragen nog steeds
      `include: ["reasoning.encrypted_content"]` aan, zodat eerder versleutelde redeneringen bij vervolgbeurten opnieuw kunnen worden afgespeeld.
    - `web_search`, `x_search` en `code_execution` worden beschikbaar gesteld als OpenClaw-tools. OpenClaw koppelt alleen de specifieke ingebouwde xAI-functie die elke tool nodig heeft aan de aanvraag van die tool, in plaats van elke native tool aan elke chatbeurt te koppelen.
    - Grok `web_search` leest `plugins.entries.xai.config.webSearch.baseUrl`.
      `x_search` leest `plugins.entries.xai.config.xSearch.baseUrl` en
      valt vervolgens terug op de basis-URL voor Grok-webzoekopdrachten.
    - `x_search` en `code_execution` worden beheerd door de meegeleverde xAI-plugin en zijn niet hardgecodeerd in de kernruntime voor modellen.
    - `code_execution` is uitvoering in een externe xAI-sandbox, niet lokale
      [`exec`](/nl/tools/exec).
  </Accordion>
</AccordionGroup>

## Live testen

De xAI-mediapaden worden gedekt door unit-tests en optionele livesuites. Exporteer
`XAI_API_KEY` in de procesomgeving voordat je liveprobes uitvoert.

```bash
pnpm test extensions/xai
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/xai.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/x-search.live.test.ts
OPENCLAW_LIVE_GATEWAY_MODELS="xai/grok-4.5,xai/grok-build-0.1,xai/grok-4.3,xai/grok-4.20-0309-reasoning,xai/grok-4.20-0309-non-reasoning" OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0 OPENCLAW_LIVE_GATEWAY_SMOKE=0 pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS=xai pnpm test:live -- test/image-generation.runtime.live.test.ts
```

Het providerspecifieke livebestand synthetiseert normale TTS en telefonievriendelijke PCM-TTS, transcribeert audio via xAI-batch-STT, streamt dezelfde PCM via xAI-realtime-STT, genereert tekst-naar-afbeelding-uitvoer en bewerkt een referentieafbeelding.
Het gedeelde livebestand voor afbeeldingen verifieert dezelfde xAI-provider via de runtimeselectie, terugval, normalisatie en het pad voor mediabijlagen van OpenClaw. De optionele Video 1.5-test dient één gegenereerde afbeelding voor het eerste frame in met 1080P en verifieert de download van de voltooide video.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Videogeneratie" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor de videotool en providerselectie.
  </Card>
  <Card title="Alle providers" href="/nl/providers/index" icon="grid-2">
    Het bredere overzicht van providers.
  </Card>
  <Card title="Problemen oplossen" href="/nl/help/troubleshooting" icon="wrench">
    Veelvoorkomende problemen en oplossingen.
  </Card>
</CardGroup>
