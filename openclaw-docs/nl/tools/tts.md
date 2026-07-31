---
read_when:
    - Tekst-naar-spraak inschakelen voor antwoorden
    - Een TTS-provider, fallbackketen of persona configureren
    - /tts-opdrachten of -instructies gebruiken
sidebarTitle: Text to speech (TTS)
summary: Tekst-naar-spraak voor uitgaande antwoorden — providers, persona's, slash-opdrachten en uitvoer per kanaal
title: Tekst-naar-spraak
x-i18n:
    generated_at: "2026-07-27T06:17:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ae9d0cc6f77c6a8b1b379c3712fd92fbbc22dae694ecdd46a0bb35cec0d29e7
    source_path: tools/tts.md
    workflow: 16
---

OpenClaw zet uitgaande antwoorden om in audio via **14 spraakproviders**:
native spraakberichten op Feishu, Matrix, Telegram en WhatsApp; audio-
bijlagen op alle andere platforms; en PCM/Ulaw-streams voor telefonie en Talk.

TTS is de spraakuitvoerhelft van de `stt-tts`-modus van Talk (`talk.speak`-gesprekken gebruiken
ditzelfde synthesepad). Providernatieve `realtime`-Talk-sessies synthetiseren
spraak binnen de realtimeprovider; `transcription`-sessies synthetiseren nooit
een gesproken antwoord van de assistent.

## Snel aan de slag

<Steps>
  <Step title="Kies een provider">
    OpenAI en ElevenLabs zijn de betrouwbaarste gehoste opties. Microsoft en
    de lokale CLI werken zonder API-sleutel. Bekijk de [providermatrix](#supported-providers)
    voor de volledige lijst.
  </Step>
  <Step title="Stel de API-sleutel in">
    Exporteer de omgevingsvariabele voor je provider (bijvoorbeeld `OPENAI_API_KEY`,
    `ELEVENLABS_API_KEY`). Microsoft en de lokale CLI hebben geen sleutel nodig.
  </Step>
  <Step title="Schakel het in de configuratie in">
    Stel `tts.auto: "always"` en `tts.provider` in:

    ```json5
    {
      tts: {
        auto: "always",
        provider: "elevenlabs",
      },
    }
    ```

  </Step>
  <Step title="Probeer het in de chat">
    `/tts status` toont de huidige status. `/tts audio Hello from OpenClaw`
    verzendt een eenmalig audioantwoord.
  </Step>
</Steps>

<Note>
Automatische TTS is standaard **uitgeschakeld**. Wanneer `tts.provider` niet is ingesteld,
kiest OpenClaw de eerste geconfigureerde provider in de automatische selecteervolgorde van het register.
De ingebouwde agenttool `tts` is alleen voor expliciete intenties: gewone chats blijven
tekst, tenzij de gebruiker om audio vraagt, `/tts` gebruikt of automatische TTS/directieve
spraak inschakelt.
</Note>

## Ondersteunde providers

| Provider          | Authenticatie                                                                                                    | Opmerkingen                                                                                         |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Azure Speech**  | `AZURE_SPEECH_KEY` + `AZURE_SPEECH_REGION` (ook `AZURE_SPEECH_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`)           | Native Ogg/Opus-uitvoer voor spraakberichten en telefonie.                                          |
| **DeepInfra**     | `DEEPINFRA_API_KEY`                                                                                               | OpenAI-compatibele TTS. Standaard ingesteld op `hexgrad/Kokoro-82M`.                                  |
| **ElevenLabs**    | `ELEVENLABS_API_KEY` of `XI_API_KEY`                                                                         | Stemklonen, meertalig, deterministisch via `seed`; gestreamd voor spraakweergave in Discord. |
| **Google Gemini** | `GEMINI_API_KEY` of `GOOGLE_API_KEY`                                                                         | Batch-TTS via de Gemini-API; personabewust via `promptTemplate: "audio-profile-v1"`.                                  |
| **Gradium**       | `GRADIUM_API_KEY`                                                                                                | Uitvoer voor spraakberichten en telefonie.                                                          |
| **Inworld**       | `INWORLD_API_KEY`                                                                                                | Streaming-TTS-API. Native Opus voor spraakberichten en PCM voor telefonie.                          |
| **Lokale CLI**    | geen                                                                                                             | Voert een geconfigureerde lokale TTS-opdracht uit.                                                  |
| **Microsoft**     | geen                                                                                                             | Openbare neurale Edge-TTS via `node-edge-tts`. Naar beste vermogen, zonder SLA.                  |
| **MiniMax**       | `MINIMAX_API_KEY` (of Token Plan: `MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`)                    | T2A v2-API. Standaard ingesteld op `speech-2.8-hd`.                                              |
| **OpenAI**        | `OPENAI_API_KEY`                                                                                                | Wordt ook gebruikt voor automatische samenvattingen; ondersteunt persona `instructions`.        |
| **OpenRouter**    | `OPENROUTER_API_KEY` (kan `models.providers.openrouter.apiKey` hergebruiken)                                                         | Standaardmodel `hexgrad/kokoro-82m`.                                                                  |
| **Volcengine**    | `VOLCENGINE_TTS_API_KEY` of `BYTEPLUS_SEED_SPEECH_API_KEY` (verouderde AppID/token: `VOLCENGINE_TTS_APPID`/`_TOKEN`)          | BytePlus Seed Speech HTTP-API.                                                                      |
| **Vydra**         | `VYDRA_API_KEY`                                                                                                | Gedeelde provider voor afbeeldingen, video en spraak.                                               |
| **xAI**           | `XAI_API_KEY`                                                                                                | Batch-TTS van xAI. Native Opus voor spraakberichten wordt **niet** ondersteund.                      |
| **Xiaomi MiMo**   | `XIAOMI_API_KEY`                                                                                                | MiMo-TTS via Xiaomi-chatvoltooiingen.                                                               |

Als meerdere providers zijn geconfigureerd, wordt de geselecteerde provider eerst gebruikt en dienen de
andere als terugvalopties. Automatische samenvattingen gebruiken `summaryModel` (of
`agents.defaults.model.primary`), dus die provider moet ook geauthenticeerd zijn
als je samenvattingen ingeschakeld laat.

<Warning>
De meegeleverde **Microsoft**-provider gebruikt de online neurale TTS-service
van Microsoft Edge via `node-edge-tts`. Het is een openbare webservice zonder gepubliceerde
SLA of quota — beschouw deze als een service naar beste vermogen. De verouderde provider-id `edge` wordt
genormaliseerd naar `microsoft` en `openclaw doctor --fix` herschrijft opgeslagen
configuratie; nieuwe configuraties moeten altijd `microsoft` gebruiken.
</Warning>

## Configuratie

De TTS-configuratie staat onder `tts` in `~/.openclaw/openclaw.json`. Kies een
voorinstelling en pas het providerblok aan. De hieronder getoonde velden `speakerVoice`/`speakerVoiceId`
zijn canoniek; de eigen veldnamen `voice`/`voiceId`/
`voiceName` van elke provider werken nog als verouderde aliassen.

<Tabs>
  <Tab title="Azure Speech">
```json5
{
  tts: {
    auto: "always",
    provider: "azure-speech",
    providers: {
      "azure-speech": {
        apiKey: "${AZURE_SPEECH_KEY}",
        region: "eastus",
        speakerVoice: "en-US-JennyNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        voiceNoteOutputFormat: "ogg-24khz-16bit-mono-opus",
      },
    },
  },
}
```
  </Tab>
  <Tab title="ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Google Gemini">
```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        apiKey: "${GEMINI_API_KEY}",
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        // Optionele stijlprompts in natuurlijke taal:
        // audioProfile: "Spreek met een kalme toon, zoals een podcastpresentator.",
        // speakerName: "Alex",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Gradium">
```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        apiKey: "${GRADIUM_API_KEY}",
        speakerVoiceId: "YTpq7expH9539ERJ",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Inworld">
```json5
{
  tts: {
    auto: "always",
    provider: "inworld",
    providers: {
      inworld: {
        apiKey: "${INWORLD_API_KEY}",
        modelId: "inworld-tts-1.5-max",
        speakerVoiceId: "Sarah",
        temperature: 0.7,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Lokale CLI">
```json5
{
  tts: {
    auto: "always",
    provider: "tts-local-cli",
    providers: {
      "tts-local-cli": {
        command: "say",
        args: ["-o", "{{OutputPath}}", "{{Text}}"],
        outputFormat: "wav",
        timeoutMs: 120000,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Microsoft (geen sleutel)">
```json5
{
  tts: {
    auto: "always",
    provider: "microsoft",
    providers: {
      microsoft: {
        enabled: true,
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        rate: "+0%",
        pitch: "+0%",
      },
    },
  },
}
```
  </Tab>
  <Tab title="MiniMax">
```json5
{
  tts: {
    auto: "always",
    provider: "minimax",
    providers: {
      minimax: {
        apiKey: "${MINIMAX_API_KEY}",
        model: "speech-2.8-hd",
        speakerVoiceId: "English_expressive_narrator",
        speed: 1.0,
        vol: 1.0,
        pitch: 0,
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI + ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "openai",
    summaryModel: "openai/gpt-4.1-mini",
    modelOverrides: { enabled: true },
    providers: {
      openai: {
        apiKey: "${OPENAI_API_KEY}",
        model: "gpt-4o-mini-tts",
        speakerVoice: "alloy",
      },
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
        voiceSettings: { stability: 0.5, similarityBoost: 0.75, style: 0.0, useSpeakerBoost: true, speed: 1.0 },
        applyTextNormalization: "auto",
        languageCode: "en",
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenRouter">
```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        apiKey: "${OPENROUTER_API_KEY}",
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Volcengine">
```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "${VOLCENGINE_TTS_API_KEY}",
        resourceId: "seed-tts-1.0",
        speakerVoice: "en_female_anna_mars_bigtts",
      },
    },
  },
}
```
  </Tab>
  <Tab title="xAI">
```json5
{
  tts: {
    auto: "always",
    provider: "xai",
    providers: {
      xai: {
        apiKey: "${XAI_API_KEY}",
        speakerVoiceId: "eve",
        language: "en",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Xiaomi MiMo">
```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "${XIAOMI_API_KEY}",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
      },
    },
  },
}
```
  </Tab>
</Tabs>

Laat voor Xiaomi `mimo-v2.5-tts-voicedesign` `speakerVoice` weg en stel `style` in op
de prompt voor het stemontwerp. OpenClaw verzendt die prompt als het TTS-bericht `user`
en verzendt `audio.voice` niet voor het voicedesign-model.

### Stemoverschrijvingen per agent

Gebruik `agents.entries.*.tts` wanneer één agent met een andere provider,
stem, model, persona of automatische TTS-modus moet spreken. Het agentblok wordt diep samengevoegd boven op
`tts`, zodat providerreferenties in de globale providerconfiguratie kunnen blijven staan:

```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: { apiKey: "${ELEVENLABS_API_KEY}", model: "eleven_multilingual_v2" },
    },
  },
  agents: {
    list: [
      {
        id: "reader",
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
      },
    ],
  },
}
```

Om een persona per agent vast te zetten, stel je `agents.entries.*.tts.persona` in naast de providerconfiguratie — deze overschrijft de algemene `tts.persona` uitsluitend voor die agent.

Volgorde van prioriteit voor automatische antwoorden, `/tts audio`, `/tts status` en de agenttool `tts`:

1. `tts`
2. actieve `agents.entries.*.tts`
3. kanaaloverschrijving, wanneer het kanaal `channels.<channel>.tts` ondersteunt
4. accountoverschrijving, wanneer het kanaal `channels.<channel>.accounts.<id>.tts` doorgeeft
5. lokale `/tts`-voorkeuren voor deze host
6. inline `[[tts:...]]`-instructies wanneer [modelgestuurde overschrijvingen](#model-driven-directives) zijn ingeschakeld

Kanaal- en accountoverschrijvingen gebruiken dezelfde vorm als `tts` en worden diep samengevoegd over de eerdere lagen, zodat gedeelde providerreferenties in `tts` kunnen blijven terwijl een kanaal- of botaccount alleen de stem van de spreker, het model, de persona of de automatische modus wijzigt:

```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { apiKey: "${OPENAI_API_KEY}", model: "gpt-4o-mini-tts" },
    },
  },
  channels: {
    feishu: {
      accounts: {
        english: {
          tts: {
            providers: {
              openai: { speakerVoice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

## Persona's

Een **persona** is een stabiele gesproken identiteit die deterministisch kan worden toegepast bij verschillende providers. De persona kan de voorkeur geven aan één provider, providerneutrale promptintentie definiëren en providerspecifieke koppelingen bevatten voor stemmen, modellen, promptsjablonen, seeds en steminstellingen.

### Minimale persona

```json5
{
  tts: {
    auto: "always",
    persona: "narrator",
    personas: {
      narrator: {
        label: "Verteller",
        provider: "elevenlabs",
        providers: {
          elevenlabs: {
            speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
            modelId: "eleven_multilingual_v2",
          },
        },
      },
    },
  },
}
```

### Volledige persona (providerspecifieke vormgeving)

```json5
{
  tts: {
    auto: "always",
    persona: "alfred",
    personas: {
      alfred: {
        label: "Alfred",
        description: "Droge, warme Britse butlerverteller.",
        provider: "google",
        fallbackPolicy: "preserve-persona",
        providers: {
          google: {
            model: "gemini-3.1-flash-tts-preview",
            speakerVoice: "Algieba",
            promptTemplate: "audio-profile-v1",
          },
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "cedar" },
          elevenlabs: {
            speakerVoiceId: "voice_id",
            modelId: "eleven_multilingual_v2",
            seed: 42,
            voiceSettings: {
              stability: 0.65,
              similarityBoost: 0.8,
              style: 0.25,
              useSpeakerBoost: true,
              speed: 0.95,
            },
          },
        },
      },
    },
  },
}
```

### Personaresolutie

De actieve persona wordt deterministisch geselecteerd:

1. `/tts persona <id>` lokale voorkeur, indien ingesteld.
2. `tts.persona`, indien ingesteld.
3. Geen persona.

Providerselectie verwerkt expliciete keuzes eerst:

1. Directe overschrijvingen (CLI, Gateway, Talk, toegestane TTS-instructies).
2. `/tts provider <id>` lokale voorkeur.
3. `provider` van de actieve persona.
4. `tts.provider`.
5. Automatische selectie uit het register.

Voor elke providerpoging voegt OpenClaw configuraties in deze volgorde samen:

1. `tts.providers.<id>`
2. `tts.personas.<persona>.providers.<id>`
3. Vertrouwde aanvraagoverschrijvingen
4. Toegestane overschrijvingen uit door het model uitgegeven TTS-instructies

### Aangepaste personavormgeving

Providerneutrale `personas.<id>.prompt.*`-configuratie is buiten gebruik gesteld. Doctor verwijdert die velden en verwijst naar de interface van de spraakprovider. Plaats ingebouwde providerinstellingen onder `personas.<id>.providers.<provider>` (bijvoorbeeld Google `personaPrompt` of OpenAI `instructions`). Voor aangepaste vormgeving implementeer je een spraakproviderplugin met `prepareSynthesis(ctx)` en retourneer je aangepaste tekst, providerconfiguratie of overschrijvingen voordat `synthesize()` wordt uitgevoerd. Hierdoor blijft de constructie van expressieve prompts in de providercode, waar de semantiek van de aanvraag bekend is.

### Terugvalbeleid

`fallbackPolicy` bepaalt het gedrag wanneer een persona **geen koppeling** heeft voor de provider die wordt geprobeerd:

| Beleid              | Gedrag                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `preserve-persona`  | **Standaard.** Providerneutrale promptvelden blijven beschikbaar; de provider kan ze gebruiken of negeren.                                            |
| `provider-defaults` | De persona wordt voor die poging weggelaten uit de promptvoorbereiding; de provider gebruikt zijn neutrale standaardwaarden terwijl de terugval naar andere providers doorgaat. |
| `fail`              | Sla die providerpoging over met `reasonCode: "not_configured"` en `personaBinding: "missing"`. Terugvalproviders worden nog steeds geprobeerd.              |

De volledige TTS-aanvraag mislukt alleen wanneer **elke** providerpoging wordt overgeslagen of mislukt.

De providerselectie voor een Talk-sessie geldt uitsluitend binnen de sessie. Een Talk-client hoort provider-ID's, model-ID's, stem-ID's en landinstellingen te kiezen uit `talk.catalog` en deze door te geven via de Talk-sessie of overdrachtsaanvraag. Het openen van een spraaksessie hoort `tts` of de algemene standaardproviders van Talk niet te wijzigen.

## Modelgestuurde instructies

Standaard **kan** de assistent `[[tts:...]]`-instructies uitvoeren om de stem, het model of de snelheid voor één antwoord te overschrijven, plus een optioneel `[[tts:text]]...[[/tts:text]]`-blok voor expressieve aanwijzingen die alleen in de audio mogen voorkomen:

```text
Alsjeblieft.

[[tts:speakerVoiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](lacht) Lees het lied nogmaals.[[/tts:text]]
```

Wanneer `tts.auto` gelijk is aan `"tagged"`, zijn **instructies vereist** om audio te activeren. Bij gestreamde bloklevering worden instructies uit zichtbare tekst verwijderd voordat het kanaal ze ontvangt, zelfs wanneer ze over aangrenzende blokken zijn verdeeld.

`provider=...` wordt genegeerd tenzij `modelOverrides.allowProvider: true`. Wanneer een antwoord `provider=...` declareert, worden de andere sleutels in die instructie uitsluitend door die provider geparseerd; niet-ondersteunde sleutels worden verwijderd en gemeld als waarschuwingen voor TTS-instructies.

**Beschikbare instructiesleutels:**

- `provider` (geregistreerde provider-ID; vereist `allowProvider: true`)
- `speakerVoice` / `speakerVoiceId` (verouderde aliassen: `voice`, `voiceName`, `voice_name`, `google_voice`, `voiceId`)
- `model` / `google_model`
- `stability`, `similarityBoost`, `style`, `speed`, `useSpeakerBoost`
- `vol` / `volume` (MiniMax-volume, `(0, 10]`)
- `pitch` (gehele MiniMax-toonhoogte, −12 tot 12; fractionele waarden worden afgekapt)
- `emotion` (Volcengine-emotietag)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

**Modeloverschrijvingen volledig uitschakelen:**

```json5
{ messages: { tts: { modelOverrides: { enabled: false } } } }
```

**Wisselen van provider toestaan terwijl andere instellingen configureerbaar blijven:**

```json5
{ messages: { tts: { modelOverrides: { enabled: true, allowProvider: true, allowSeed: false } } } }
```

## Slash-opdrachten

Eén opdracht: `/tts`. Op Discord registreert OpenClaw ook `/voice`, omdat `/tts` een ingebouwde Discord-opdracht is — de tekst `/tts ...` werkt nog steeds.

```text
/tts off | on | status
/tts chat on | off | default
/tts latest
/tts provider <id>
/tts persona <id> | off
/tts limit <chars>
/tts summary off
/tts audio <text>
```

<Note>
Opdrachten vereisen een geautoriseerde afzender (regels voor toelatingslijsten/eigenaren zijn van toepassing) en `commands.text` of registratie van systeemeigen opdrachten moet zijn ingeschakeld.
</Note>

Gedragsnotities:

- `/tts on` schrijft de lokale TTS-voorkeur naar `always`; `/tts off` schrijft deze naar `off`.
- `/tts chat on|off|default` schrijft een sessiegebonden automatische TTS-overschrijving voor de huidige chat.
- `/tts persona <id>` schrijft de lokale personavoorkeur; `/tts persona off` wist deze.
- `/tts latest` leest het nieuwste assistentantwoord uit het transcript van de huidige sessie en verzendt het eenmaal als audio. Alleen een hash van dat antwoord wordt in het sessie-item opgeslagen om dubbele spraakverzendingen te voorkomen.
- `/tts audio` genereert een eenmalig audioantwoord (schakelt TTS **niet** in).
- `/tts limit <chars>` accepteert **100–4096** (4096 is het maximum voor Telegram-bijschriften/-berichten); waarden buiten dat bereik worden geweigerd.
- `limit` en `summary` worden opgeslagen in **lokale voorkeuren**, niet in de hoofdconfiguratie.
- `/tts status` bevat terugvaldiagnostiek voor de nieuwste poging — `Fallback: <primary> -> <used>`, `Attempts: ...` en details per poging (`provider:outcome(reasonCode) latency`).
- `/status` toont de actieve TTS-modus plus de geconfigureerde provider, het model, de stem en opgeschoonde metagegevens van aangepaste eindpunten wanneer TTS is ingeschakeld.

## Voorkeuren per gebruiker

Slash-opdrachten schrijven lokale overschrijvingen naar het pad voor TTS-voorkeuren. De standaardwaarde is `~/.openclaw/settings/tts.json`; overschrijf deze met `OPENCLAW_TTS_PREFS`. Doctor verplaatst de buiten gebruik gestelde algemene waarde `tts.prefsPath` naar gedeelde machinestatus. Geavanceerde configuraties met meerdere agents kunnen nog steeds `agents.entries.<id>.tts.prefsPath` instellen wanneer agents bewust afzonderlijke voorkeursopslag gebruiken.

| Opgeslagen veld | Effect                                                                           |
| ------------ | -------------------------------------------------------------------------------- |
| `auto`       | Lokale automatische TTS-overschrijving (`always`, `off`, …)                                     |
| `provider`   | Lokale overschrijving van de primaire provider                                                  |
| `persona`    | Lokale personaoverschrijving                                                           |
| `maxLength`  | Drempelwaarde voor samenvatting/afkapping (standaard `1500` tekens, `/tts limit` bereik 100–4096) |
| `summarize`  | Schakelaar voor samenvatting (standaard `true`)                                                  |

Deze overschrijven de effectieve configuratie van `tts` plus het actieve `agents.entries.*.tts`-blok voor die host.

## Uitvoerformaten

De levering van TTS-spraak wordt bepaald door de kanaalmogelijkheden. Kanaalplugins geven aan of TTS in spraakstijl providers om een systeemeigen `voice-note`-doel moet vragen of normale `audio-file`-synthese moet behouden, en of het kanaal niet-systeemeigen uitvoer vóór verzending transcodeert.

| Doel                                  | Indeling                                                                                                                                |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Feishu / Matrix / Telegram / WhatsApp | Antwoorden als spraakbericht geven de voorkeur aan **Opus** (`opus_48000_64` van ElevenLabs, `opus` van OpenAI). 48 kHz / 64 kbps biedt een goede balans tussen helderheid en grootte. |
| Andere kanalen                        | **MP3** (`mp3_44100_128` van ElevenLabs, `mp3` van OpenAI). 44.1 kHz / 128 kbps is de standaardbalans voor spraak.                  |
| Talk / telefonie                      | Providerspecifieke **PCM** (Inworld 22050 Hz, Google 24 kHz), of `ulaw_8000` van Gradium voor telefonie.                                 |

Opmerkingen per provider:

- **Transcodering voor Feishu / WhatsApp:** wanneer een antwoord als spraakbericht binnenkomt als MP3/WebM/WAV/M4A of een ander waarschijnlijk audiobestand, transcodeert de kanaalplugin het vóór verzending als systeemeigen spraakbericht naar 48 kHz Ogg/Opus met `ffmpeg` (`libopus`, 64 kbps). WhatsApp verzendt het resultaat via de Baileys-`audio`-payload met `ptt: true` en `audio/ogg; codecs=opus`. Bij een transcoderingsfout: Feishu vangt de fout op en valt terug op verzending van het oorspronkelijke bestand als gewone bijlage; WhatsApp heeft geen terugvaloptie, waardoor de verzending zelf mislukt in plaats van een incompatibele PTT-payload te plaatsen.
- **MiniMax:** MP3 (`speech-2.8-hd`-model, samplefrequentie van 32 kHz) voor normale audiobijlagen; getranscodeerd naar 48 kHz Opus met `ffmpeg` voor doelen die door het kanaal als spraakbericht worden aangeduid.
- **Xiaomi MiMo:** standaard MP3, of WAV wanneer dit is geconfigureerd; getranscodeerd naar 48 kHz Opus met `ffmpeg` voor doelen die door het kanaal als spraakbericht worden aangeduid.
- **Lokale CLI:** gebruikt de geconfigureerde `outputFormat`. Doelen voor spraakberichten worden geconverteerd naar Ogg/Opus en telefonie-uitvoer wordt met `ffmpeg` geconverteerd naar onbewerkte 16 kHz mono-PCM.
- **Google Gemini:** retourneert onbewerkte 24 kHz PCM. OpenClaw verpakt deze als WAV voor audiobijlagen, transcodeert deze naar 48 kHz Opus voor doelen voor spraakberichten en retourneert PCM rechtstreeks voor Talk/telefonie.
- **Gradium:** WAV voor audiobijlagen, Opus voor doelen voor spraakberichten en `ulaw_8000` op 8 kHz voor telefonie.
- **Inworld:** MP3 voor normale audiobijlagen, systeemeigen `OGG_OPUS` voor doelen voor spraakberichten en onbewerkte `PCM` op 22050 Hz voor Talk/telefonie.
- **xAI:** standaard MP3; synthese van audiobestanden kan `mp3`, `wav`, `pcm`, `mulaw` of `alaw` gebruiken voor zowel gebufferde als gestreamde uitvoer. Doelen voor spraakberichten gebruiken MP3 voor streaming en als gebufferde terugvaloptie, omdat de uitvoer van xAI voor `pcm`, `mulaw` en `alaw` onbewerkte audio zonder headers is. Gebufferde synthese gebruikt het batch-REST-`/v1/tts`-eindpunt van xAI; `textToSpeechStream` gebruikt systeemeigen `wss://api.x.ai/v1/tts`. Dit is niet het realtime-spraakcontract. Een systeemeigen Opus-indeling voor spraakberichten wordt niet ondersteund.
- **Microsoft:** gebruikt `microsoft.outputFormat` (standaard `audio-24khz-48kbitrate-mono-mp3`).
  - Het meegeleverde transport accepteert een `outputFormat`, maar niet alle indelingen zijn beschikbaar via de service.
  - Waarden voor de uitvoerindeling volgen de uitvoerindelingen van Microsoft Speech (waaronder Ogg/WebM Opus).
  - Telegram `sendVoice` accepteert OGG/MP3/M4A; gebruik OpenAI/ElevenLabs als je gegarandeerde Opus-spraakberichten nodig hebt.
  - Als de geconfigureerde Microsoft-uitvoerindeling mislukt, probeert OpenClaw het opnieuw met MP3.
  - Wanneer geen expliciete stemoverschrijving is ingesteld en de standaard Engelse stem wordt gebruikt, schakelt OpenClaw automatisch over naar een Chinese neurale stem (`zh-CN-XiaoxiaoNeural`, landinstelling `zh-CN`) als de antwoordtekst voornamelijk uit CJK-tekens bestaat.

De uitvoerindelingen van OpenAI en ElevenLabs liggen per kanaal vast zoals hierboven vermeld.

## Gedrag van automatische TTS

Wanneer `tts.auto` is ingeschakeld, doet OpenClaw het volgende:

- Slaat TTS over als het antwoord al gestructureerde media bevat.
- Slaat zeer korte antwoorden over (minder dan 10 tekens).
- Vat lange antwoorden samen wanneer samenvattingen zijn ingeschakeld, met
  `summaryModel` (of `agents.defaults.model.primary`).
- Voegt de gegenereerde audio toe aan het antwoord.
- Verzendt in `mode: "final"` nog steeds uitsluitend audio-TTS voor gestreamde definitieve antwoorden
  nadat de tekststream is voltooid; de gegenereerde media ondergaan dezelfde
  normalisatie van kanaalmedia als normale antwoordbijlagen.

Als het antwoord langer is dan `maxLength`, slaat OpenClaw audio nooit volledig over:

- **Samenvatting aan** (standaard) en er is een samenvattingsmodel beschikbaar: vat de
  tekst samen tot ongeveer `maxLength` tekens en synthetiseert vervolgens de samenvatting.
- **Samenvatting uit**, samenvatten mislukt of er is geen API-sleutel beschikbaar voor het
  samenvattingsmodel: kort de tekst in tot `maxLength` tekens en synthetiseert de
  ingekorte tekst.

```text
Antwoord -> TTS ingeschakeld?
  nee -> tekst verzenden
  ja  -> bevat media / kort?
          ja  -> tekst verzenden
          nee -> lengte > limiet?
                   nee -> TTS -> audio toevoegen
                   ja  -> samenvatting ingeschakeld en beschikbaar?
                            nee -> inkorten -> TTS -> audio toevoegen
                            ja  -> samenvatten -> TTS -> audio toevoegen
```

## Veldreferentie

<AccordionGroup>
  <Accordion title="TTS op het hoogste niveau.*">
    <ParamField path="auto" type='"off" | "always" | "inbound" | "tagged"'>
      Modus voor automatische TTS. `inbound` verzendt alleen audio na een inkomend spraakbericht; `tagged` verzendt alleen audio wanneer het antwoord `[[tts:...]]`-instructies of een `[[tts:text]]`-blok bevat.
    </ParamField>
    <ParamField path="enabled" type="boolean" deprecated>
      Verouderde schakelaar. `openclaw doctor --fix` migreert deze naar `auto`.
    </ParamField>
    <ParamField path="mode" type='"final" | "all"' default="final">
      `"all"` neemt naast definitieve antwoorden ook tool-/blokantwoorden op.
    </ParamField>
    <ParamField path="provider" type="string">
      Id van de spraakprovider. Wanneer deze niet is ingesteld, gebruikt OpenClaw de eerste geconfigureerde provider in de automatische selecteervolgorde van het register. Verouderde `provider: "edge"` wordt door `openclaw doctor --fix` herschreven naar `"microsoft"`.
    </ParamField>
    <ParamField path="persona" type="string">
      Actieve persona-id uit `personas`. Genormaliseerd naar kleine letters.
    </ParamField>
    <ParamField path="personas.<id>" type="object">
      Stabiele gesproken identiteit. Velden: `label`, `description`, `provider`, `fallbackPolicy`, `prompt`, `providers.<provider>`. Zie [Persona's](#personas).
    </ParamField>
    <ParamField path="summaryModel" type="string">
      Goedkoop model voor automatische samenvatting; standaard `agents.defaults.model.primary`. Accepteert `provider/model` of een geconfigureerde modelalias.
    </ParamField>
    <ParamField path="modelOverrides" type="object">
      Staat toe dat het model TTS-instructies uitvoert. `enabled` is standaard `true`; `allowProvider` is standaard `false`.
    </ParamField>
    <ParamField path="providers.<id>" type="object">
      Instellingen die eigendom zijn van de provider, geïndexeerd op spraakprovider-id. Verouderde directe blokken (`tts.openai`, `.elevenlabs`, `.microsoft`, `.edge`) worden herschreven door `openclaw doctor --fix`; leg alleen `tts.providers.<id>` vast.
    </ParamField>
    <ParamField path="maxTextLength" type="number" default="4096">
      Harde limiet voor het aantal invoertekens voor TTS. `/tts audio`, `tts.convert` en `tts.speak` mislukken als deze wordt overschreden.
    </ParamField>
    <ParamField path="timeoutMs" type="number" default="30000">
      Time-out van aanvragen in milliseconden. Een `timeoutMs` per aanroep (agenttool, Gateway) heeft voorrang wanneer deze is ingesteld; anders heeft een expliciet geconfigureerde `tts.timeoutMs` voorrang op elke door een plugin ingestelde standaardwaarde van de provider.
    </ParamField>
  </Accordion>

Provider-`apiKey`-velden kunnen onbewerkte tekenreeksen of SecretRefs zijn. Als tijdens een koude start van de Gateway
een TTS-SecretRef niet beschikbaar is, wordt de ingebouwde TTS-mogelijkheid
gemarkeerd als geconfigureerd-maar-niet-beschikbaar in plaats van de Gateway te stoppen. `tts.speak` retourneert dan
`UNAVAILABLE` met reden `SECRET_SURFACE_UNAVAILABLE` en er wordt geen providerverzoek
verzonden. Status en doctor vermelden de gedegradeerde TTS-eigenaar en de bijbehorende configuratiepaden. De
expliciete verwijzingen blijven in de runtime-snapshot staan, zodat omgevings- of profielreferenties
niet ongemerkt een ander account kunnen selecteren. Herlaadbewerkingen en controles voorafgaand aan het schrijven van configuratie
passen het degradatiebeleid met eigenaarsbesef toe: een ongewijzigde, geschikte TTS-
eigenaar mag de laatst bekende werkende referenties als verouderd behouden, terwijl een nieuwe of gewijzigde
fout koud wordt zonder gezonde eigenaren te blokkeren. Structureel ongeldige verwijzingen
en opgeloste waarden laten het opstarten nog steeds mislukken of zorgen ervoor dat de update wordt geweigerd.

  <Accordion title="Azure Speech">
    <ParamField path="apiKey" type="string">Omgeving: `AZURE_SPEECH_KEY`, `AZURE_SPEECH_API_KEY` of `SPEECH_KEY`.</ParamField>
    <ParamField path="region" type="string">Azure Speech-regio (bijv. `eastus`). Omgeving: `AZURE_SPEECH_REGION` of `SPEECH_REGION`.</ParamField>
    <ParamField path="endpoint" type="string">Optionele overschrijving van het Azure Speech-eindpunt (alias `baseUrl`).</ParamField>
    <ParamField path="speakerVoice" type="string">ShortName van de Azure-stem. Standaard `en-US-JennyNeural`. Verouderde alias: `voice`.</ParamField>
    <ParamField path="lang" type="string">SSML-taalcode. Standaard `en-US`.</ParamField>
    <ParamField path="outputFormat" type="string">Azure `X-Microsoft-OutputFormat` voor standaardaudio. Standaard `audio-24khz-48kbitrate-mono-mp3`.</ParamField>
    <ParamField path="voiceNoteOutputFormat" type="string">Azure `X-Microsoft-OutputFormat` voor uitvoer van spraakberichten. Standaard `ogg-24khz-16bit-mono-opus`.</ParamField>
  </Accordion>

  <Accordion title="ElevenLabs">
    <ParamField path="apiKey" type="string">Valt terug op `ELEVENLABS_API_KEY` of `XI_API_KEY`.</ParamField>
    <ParamField path="model" type="string">Model-id. Standaard `eleven_multilingual_v2`. Verouderde id's `eleven_turbo_v2_5`/`eleven_turbo_v2` worden genormaliseerd naar het overeenkomende `flash`-model.</ParamField>
    <ParamField path="speakerVoiceId" type="string">ElevenLabs-stem-id. Standaard `pMsXgVXv3BLzUgSXRplE`. Verouderde alias: `voiceId`.</ParamField>
    <ParamField path="voiceSettings" type="object">
      `stability`, `similarityBoost`, `style` (elk `0..1`, standaardwaarden `0.5`/`0.75`/`0`), `useSpeakerBoost` (`true|false`, standaard `true`), `speed` (`0.5..2.0`, standaard `1.0`).
    </ParamField>
    <ParamField path="applyTextNormalization" type='"auto" | "on" | "off"'>Modus voor tekstnormalisatie.</ParamField>
    <ParamField path="languageCode" type="string">2-letterige ISO 639-1-code (bijv. `en`, `de`).</ParamField>
    <ParamField path="seed" type="number">Geheel getal `0..4294967295` voor determinisme op basis van beste inspanning.</ParamField>
    <ParamField path="baseUrl" type="string">Overschrijf de basis-URL van de ElevenLabs-API.</ParamField>
  </Accordion>

  <Accordion title="Google Gemini">
    <ParamField path="apiKey" type="string">Valt terug op `GEMINI_API_KEY` / `GOOGLE_API_KEY`. Indien weggelaten, kan TTS `models.providers.google.apiKey` hergebruiken voordat op de omgevingsvariabele wordt teruggevallen.</ParamField>
    <ParamField path="model" type="string">Gemini TTS-model. Standaard `gemini-3.1-flash-tts-preview`.</ParamField>
    <ParamField path="speakerVoice" type="string">Naam van een vooraf ingebouwde Gemini-stem. Standaard `Kore`. Verouderde aliassen: `voiceName`, `voice`.</ParamField>
    <ParamField path="audioProfile" type="string">Stijlprompt in natuurlijke taal die vóór de gesproken tekst wordt geplaatst.</ParamField>
    <ParamField path="speakerName" type="string">Optioneel sprekerlabel dat vóór de gesproken tekst wordt geplaatst wanneer je prompt een benoemde spreker gebruikt.</ParamField>
    <ParamField path="promptTemplate" type='"audio-profile-v1"'>Stel in op `audio-profile-v1` om actieve personapromptvelden in een deterministische Gemini TTS-promptstructuur te verpakken.</ParamField>
    <ParamField path="personaPrompt" type="string">Aanvullende Google-specifieke personaprompttekst die aan de Director's Notes van de sjabloon wordt toegevoegd.</ParamField>
    <ParamField path="baseUrl" type="string">Alleen `https://generativelanguage.googleapis.com` wordt geaccepteerd.</ParamField>
  </Accordion>

  <Accordion title="Gradium">
    <ParamField path="apiKey" type="string">Omgevingsvariabele: `GRADIUM_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">HTTPS-URL van de Gradium-API op `api.gradium.ai`. Standaard `https://api.gradium.ai`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">Standaard Emma (`YTpq7expH9539ERJ`). Verouderde alias: `voiceId`.</ParamField>
  </Accordion>

  <Accordion title="Inworld">
    ### Primaire Inworld-configuratie

    <ParamField path="apiKey" type="string">Omgevingsvariabele: `INWORLD_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">Standaard `https://api.inworld.ai`.</ParamField>
    <ParamField path="modelId" type="string">Standaard `inworld-tts-1.5-max`. Ook: `inworld-tts-1.5-mini`, `inworld-tts-1-max`, `inworld-tts-1`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">Standaard `Sarah`. Verouderde alias: `voiceId`.</ParamField>
    <ParamField path="temperature" type="number">Samplingtemperatuur `0..2` (0 uitgesloten).</ParamField>

  </Accordion>

  <Accordion title="Lokale CLI (tts-local-cli)">
    <ParamField path="command" type="string">Lokaal uitvoerbaar bestand of opdrachttekenreeks voor CLI-TTS.</ParamField>
    <ParamField path="args" type="string[]">Opdrachtargumenten. Ondersteunt de tijdelijke aanduidingen `{{Text}}`, `{{OutputPath}}`, `{{OutputDir}}`, `{{OutputBase}}`.</ParamField>
    <ParamField path="outputFormat" type='"mp3" | "opus" | "wav"'>Verwachte CLI-uitvoerindeling. Standaard `mp3` voor audiobijlagen.</ParamField>
    <ParamField path="timeoutMs" type="number">Time-out van de opdracht in milliseconden. Standaard `120000`.</ParamField>
    <ParamField path="cwd" type="string">Optionele werkmap voor de opdracht.</ParamField>
    <ParamField path="env" type="Record<string, string>">Optionele overschrijvingen van omgevingsvariabelen voor de opdracht.</ParamField>

    De standaarduitvoer van de opdracht en gegenereerde of geconverteerde audio zijn beperkt tot 50 MiB. Diagnostische standaardfoutuitvoer is beperkt tot 1 MiB. OpenClaw beëindigt de opdracht en laat de synthese mislukken wanneer een van beide limieten wordt overschreden.

  </Accordion>

  <Accordion title="Microsoft (geen API-sleutel)">
    <ParamField path="enabled" type="boolean" default="true">Gebruik van Microsoft-spraak toestaan.</ParamField>
    <ParamField path="speakerVoice" type="string">Naam van de neurale Microsoft-stem (bijv. `en-US-MichelleNeural`). Verouderde alias: `voice`. Als de standaard Engelse stem actief is en de antwoordtekst hoofdzakelijk uit CJK-tekens bestaat, schakelt OpenClaw automatisch over naar `zh-CN-XiaoxiaoNeural`.</ParamField>
    <ParamField path="lang" type="string">Taalcode (bijv. `en-US`).</ParamField>
    <ParamField path="outputFormat" type="string">Microsoft-uitvoerindeling. Standaard `audio-24khz-48kbitrate-mono-mp3`. Niet alle indelingen worden ondersteund door het meegeleverde, op Edge gebaseerde transport.</ParamField>
    <ParamField path="rate / pitch / volume" type="string">Percentagetekenreeksen (bijv. `+10%`, `-5%`).</ParamField>
    <ParamField path="saveSubtitles" type="boolean">JSON-ondertitels naast het audiobestand schrijven.</ParamField>
    <ParamField path="proxy" type="string">Proxy-URL voor Microsoft-spraakverzoeken.</ParamField>
    <ParamField path="timeoutMs" type="number">Overschrijving van de time-out voor verzoeken (ms).</ParamField>
    <ParamField path="edge.*" type="object" deprecated>Verouderde alias. Voer `openclaw doctor --fix` uit om de opgeslagen configuratie te herschrijven naar `providers.microsoft`.</ParamField>
  </Accordion>

  <Accordion title="MiniMax">
    <ParamField path="apiKey" type="string">Valt terug op `MINIMAX_API_KEY`. Authenticatie met Token Plan via `MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY` of `MINIMAX_CODING_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">Standaard `https://api.minimax.io`. Omgevingsvariabele: `MINIMAX_API_HOST`.</ParamField>
    <ParamField path="model" type="string">Standaard `speech-2.8-hd`. Omgevingsvariabele: `MINIMAX_TTS_MODEL`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">Standaard `English_expressive_narrator`. Omgevingsvariabele: `MINIMAX_TTS_VOICE_ID`. Verouderde alias: `voiceId`.</ParamField>
    <ParamField path="speed" type="number">`0.5..2.0`. Standaard `1.0`.</ParamField>
    <ParamField path="vol" type="number">`(0, 10]`. Standaard `1.0`.</ParamField>
    <ParamField path="pitch" type="number">Geheel getal `-12..12`. Standaard `0`. Fractionele waarden worden vóór het verzoek afgekapt.</ParamField>
  </Accordion>

  <Accordion title="OpenAI">
    <ParamField path="apiKey" type="string">Valt terug op `OPENAI_API_KEY`.</ParamField>
    <ParamField path="model" type="string">OpenAI TTS-model-id. Standaard `gpt-4o-mini-tts`.</ParamField>
    <ParamField path="speakerVoice" type="string">Stemnaam (bijv. `alloy`, `cedar`). Standaard `coral`. Verouderde alias: `voice`.</ParamField>
    <ParamField path="instructions" type="string">Expliciet OpenAI-veld `instructions`. Wanneer dit is ingesteld, worden personapromptvelden **niet** automatisch toegewezen.</ParamField>
    <ParamField path="extraBody / extra_body" type="Record<string, unknown>">Extra JSON-velden die na de gegenereerde OpenAI TTS-velden worden samengevoegd met de hoofdtekst van `/audio/speech`-verzoeken. Gebruik dit voor OpenAI-compatibele eindpunten zoals Kokoro die providerspecifieke sleutels zoals `lang` vereisen; onveilige prototypesleutels worden genegeerd.</ParamField>
    <ParamField path="baseUrl" type="string">
      Overschrijf het OpenAI TTS-eindpunt. Resolutievolgorde: configuratie → `OPENAI_TTS_BASE_URL` → `https://api.openai.com/v1`. Niet-standaardwaarden worden behandeld als OpenAI-compatibele TTS-eindpunten, zodat aangepaste model- en stemnamen worden geaccepteerd en `speed` de bereikcontrole voor `0.25..4.0` verliest.
    </ParamField>
  </Accordion>

  <Accordion title="OpenRouter">
    <ParamField path="apiKey" type="string">Omgevingsvariabele: `OPENROUTER_API_KEY`. Kan `models.providers.openrouter.apiKey` hergebruiken.</ParamField>
    <ParamField path="baseUrl" type="string">Standaard `https://openrouter.ai/api/v1`. Verouderde `https://openrouter.ai/v1` wordt genormaliseerd.</ParamField>
    <ParamField path="model" type="string">Standaard `hexgrad/kokoro-82m`. Alias: `modelId`.</ParamField>
    <ParamField path="speakerVoice" type="string">Standaard `af_alloy`. Verouderde aliassen: `voice`, `voiceId`.</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "pcm"'>Standaard `mp3`.</ParamField>
    <ParamField path="speed" type="number">Providerspecifieke overschrijving van de snelheid.</ParamField>
  </Accordion>

  <Accordion title="Volcengine (BytePlus Seed Speech)">
    <ParamField path="apiKey" type="string">Omgevingsvariabele: `VOLCENGINE_TTS_API_KEY` of `BYTEPLUS_SEED_SPEECH_API_KEY`.</ParamField>
    <ParamField path="resourceId" type="string">Standaard `seed-tts-1.0`. Omgevingsvariabele: `VOLCENGINE_TTS_RESOURCE_ID`. Gebruik `seed-tts-2.0` wanneer je project recht heeft op TTS 2.0.</ParamField>
    <ParamField path="appKey" type="string">App-sleutelheader. Standaard `aGjiRDfUWi`. Omgevingsvariabele: `VOLCENGINE_TTS_APP_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">Overschrijf het HTTP-eindpunt voor Seed Speech TTS. Omgevingsvariabele: `VOLCENGINE_TTS_BASE_URL`.</ParamField>
    <ParamField path="speakerVoice" type="string">Stemtype. Standaard `en_female_anna_mars_bigtts`. Omgevingsvariabele: `VOLCENGINE_TTS_VOICE`. Verouderde alias: `voice`.</ParamField>
    <ParamField path="speedRatio" type="number">Providerspecifieke snelheidsverhouding, `0.2..3`.</ParamField>
    <ParamField path="emotion" type="string">Providerspecifieke emotietag.</ParamField>
    <ParamField path="appId / token / cluster" type="string" deprecated>Verouderde velden van de Volcengine Speech Console. Omgevingsvariabelen: `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOLCENGINE_TTS_CLUSTER` (standaard `volcano_tts`).</ParamField>
  </Accordion>

  <Accordion title="xAI">
    <ParamField path="apiKey" type="string">Omgevingsvariabele: `XAI_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">Standaard `https://api.x.ai/v1`. Omgevingsvariabele: `XAI_BASE_URL`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">Standaard `eve`. Met authenticatie haalt `openclaw infer tts voices --provider xai` de huidige ingebouwde catalogus op; zonder authenticatie geeft het de offline terugvalopties `ara`, `eve`, `leo`, `rex` en `sal` weer. Aangepaste stem-id's van het account worden doorgestuurd, zelfs wanneer ze niet in de ingebouwde lijst voorkomen. Verouderde alias: `voiceId`.</ParamField>
    <ParamField path="language" type="string">BCP-47-taalcode of `auto`. Standaard `en`.</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "wav" | "pcm" | "mulaw" | "alaw"'>Standaard `mp3`.</ParamField>
    <ParamField path="speed" type="number">Providerspecifieke overschrijving van de snelheid, `0.7..1.5`.</ParamField>
  </Accordion>

  <Accordion title="Xiaomi MiMo">
    <ParamField path="apiKey" type="string">Omgevingsvariabele: `XIAOMI_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">Standaard `https://api.xiaomimimo.com/v1`. Omgevingsvariabele: `XIAOMI_BASE_URL`.</ParamField>
    <ParamField path="model" type="string">Standaard `mimo-v2.5-tts`. Omgevingsvariabele: `XIAOMI_TTS_MODEL`. Ondersteunt ook `mimo-v2.5-tts-voicedesign`.</ParamField>
    <ParamField path="speakerVoice" type="string">Standaard `mimo_default` voor modellen met vooraf ingestelde stemmen. Omgevingsvariabele: `XIAOMI_TTS_VOICE`. Verouderde alias: `voice`. Wordt niet verzonden voor `mimo-v2.5-tts-voicedesign`.</ParamField>
    <ParamField path="format" type='"mp3" | "wav"'>Standaard `mp3`. Omgevingsvariabele: `XIAOMI_TTS_FORMAT`.</ParamField>
    <ParamField path="style" type="string">Optionele stijlinstructie in natuurlijke taal die als gebruikersbericht wordt verzonden en niet wordt uitgesproken. Voor `mimo-v2.5-tts-voicedesign` is dit de prompt voor stemontwerp; OpenClaw levert een standaardwaarde wanneer deze wordt weggelaten.</ParamField>
  </Accordion>
</AccordionGroup>

## Agenttool

De tool `tts` zet tekst om in spraak en retourneert een audiobijlage voor
het afleveren van antwoorden. Op Feishu, Matrix, Telegram en WhatsApp wordt de audio
afgeleverd als een spraakbericht in plaats van als bestandsbijlage. Feishu en
WhatsApp kunnen niet-Opus TTS-uitvoer op dit pad transcoderen wanneer `ffmpeg`
beschikbaar is.

WhatsApp verzendt audio via Baileys als een PTT-spraakbericht (`audio` met
`ptt: true`) en verzendt zichtbare tekst **afzonderlijk** van PTT-audio, omdat
clients bijschriften bij spraakberichten niet consistent weergeven.

De tool accepteert optionele velden `channel` en `timeoutMs`; `timeoutMs` is een
time-out per aanroep voor providerverzoeken in milliseconden. Waarden per aanroep overschrijven
`tts.timeoutMs`; geconfigureerde TTS-time-outs overschrijven elke door een Plugin ingestelde
standaardwaarde van de provider.

## Gateway-RPC

| Methode           | Doel                                         |
| ----------------- | -------------------------------------------- |
| `tts.status`      | Huidige TTS-status en laatste poging lezen.  |
| `tts.enable`      | Lokale automatische voorkeur instellen op `always`. |
| `tts.disable`     | Lokale automatische voorkeur instellen op `off`. |
| `tts.convert`     | Eenmalig tekst → audio.                      |
| `tts.setProvider` | Lokale providervoorkeur instellen.           |
| `tts.personas`    | Geconfigureerde persona's en de actieve persona weergeven. |
| `tts.setPersona`  | Lokale personavoorkeur instellen.            |
| `tts.providers`   | Geconfigureerde providers en hun status weergeven. |

## Servicekoppelingen

- [OpenAI-handleiding voor tekst-naar-spraak](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI Audio API-referentie](https://platform.openai.com/docs/api-reference/audio)
- [Azure Speech REST voor tekst-naar-spraak](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech)
- [Azure Speech-provider](/nl/providers/azure-speech)
- [ElevenLabs tekst-naar-spraak](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs-authenticatie](https://elevenlabs.io/docs/api-reference/authentication)
- [Gradium](/nl/providers/gradium)
- [Inworld TTS API](https://docs.inworld.ai/tts/tts)
- [MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [Volcengine TTS HTTP API](/nl/providers/volcengine#text-to-speech)
- [Xiaomi MiMo-spraaksynthese](/nl/providers/xiaomi#text-to-speech)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft Speech-uitvoerindelingen](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)
- [xAI tekst-naar-spraak](https://docs.x.ai/developers/rest-api-reference/inference/voice#text-to-speech-rest)

## Gerelateerd

- [Mediaoverzicht](/nl/tools/media-overview)
- [Muziek genereren](/nl/tools/music-generation)
- [Video genereren](/nl/tools/video-generation)
- [Slash-opdrachten](/nl/tools/slash-commands)
- [Plugin voor spraakoproepen](/nl/plugins/voice-call)
