---
read_when:
    - Media-inzicht ontwerpen of herstructureren
    - Afstellen van de voorverwerking van inkomende audio, video en afbeeldingen
sidebarTitle: Media understanding
summary: Inkomend begrip van afbeeldingen/audio/video (optioneel) met provider- en CLI-fallbacks
title: Mediabegrip
x-i18n:
    generated_at: "2026-07-27T06:20:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 38e9a0f89607bb9c4af85689ef0fbd3df9234b36e06d86c129e0d823d6e05143
    source_path: nodes/media-understanding.md
    workflow: 16
---

OpenClaw kan binnenkomende media (afbeelding/audio/video) samenvatten voordat de antwoordpijplijn wordt uitgevoerd, zodat opdrachtparsing en routering werken met korte tekst in plaats van onbewerkte bytes. Begrip detecteert automatisch lokale tools of providersleutels, of je kunt expliciete modellen configureren. De oorspronkelijke media worden zoals gewoonlijk altijd aan het model geleverd; wanneer begrip mislukt of is uitgeschakeld, gaat de antwoordflow ongewijzigd verder.

Leveranciersplugins registreren metagegevens over mogelijkheden (welke provider welk mediatype ondersteunt, standaardmodel, prioriteit). De kern van OpenClaw beheert de gedeelde `tools.media`-configuratie, terugvalvolgorde en integratie met de antwoordpijplijn.

## Hoe het werkt

<Steps>
  <Step title="Bijlagen verzamelen">
    Verzamel geordende feiten over binnenkomende media (`path`, `url`, `contentType` en `kind`).
  </Step>
  <Step title="Per mogelijkheid selecteren">
    Selecteer voor elke ingeschakelde mogelijkheid (afbeelding/audio/video) bijlagen volgens het `attachments`-beleid (standaard: alleen de eerste bijlage).
  </Step>
  <Step title="Een model kiezen">
    Kies de eerste geschikte modelvermelding (grootte + mogelijkheid + beschikbare authenticatie).
  </Step>
  <Step title="Terugvallen bij fouten">
    Als een model een fout geeft, een time-out bereikt of de media groter zijn dan `maxBytes`, probeer je de volgende vermelding.
  </Step>
  <Step title="Toepassen bij succes">
    `Body` wordt een `[Image]`-, `[Audio]`- of `[Video]`-blok. Audio stelt ook `{{Transcript}}` in; opdrachtparsing gebruikt bij aanwezigheid de bijschrifttekst en anders het transcript. Bijschriften blijven binnen het blok behouden als `User text:`.
  </Step>
</Steps>

## Configuratie

`tools.media` bevat één modelleerlijst met mogelijkheidstags en enkele kleine bedieningselementen per mogelijkheid:

```json5
{
  tools: {
    media: {
      concurrency: 2, // maximaal aantal gelijktijdige uitvoeringen van mogelijkheden (standaard)
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["image", "video"] },
      ],
      image: { preferredModel: "google/gemini-3-flash-preview" },
      audio: { enabled: true },
      video: { enabled: true },
    },
  },
}
```

Sleutels per mogelijkheid (`image`/`audio`/`video`):

| Sleutel           | Type      | Standaard                              | Opmerkingen                                                          |
| ---------------- | --------- | -------------------------------------- | -------------------------------------------------------------------- |
| `enabled`        | `boolean` | automatisch (`false` schakelt uit)        | Stel `false` in om automatische detectie voor deze mogelijkheid uit te schakelen |
| `preferredModel` | `string`  | eerste compatibele vermelding          | Geef voorkeur aan `provider/model`, model-id, `provider:<id>` of `cli:command` |
| `prompt`         | `string`  | standaard voor de mogelijkheid         | Standaardprompt wanneer een vermelding deze niet overschrijft        |
| `maxChars`       | `number`  | `500` afbeelding/video, niet ingesteld voor audio | Standaardlimiet voor uitvoer                                          |
| `maxBytes`       | `number`  | 10MB afbeelding, 20MB audio, 50MB video | Standaardlimiet voor invoer                                          |
| `timeoutSeconds` | `number`  | `60` afbeelding/audio, `120` video | Standaardtime-out voor aanvragen                                     |
| `language`       | `string`  | niet ingesteld                         | Hint voor audiotranscriptie                                          |
| `scope`          | object    | niet ingesteld                         | Beperken op kanaal-/chattype-/bronsleutel                            |
| `attachments`    | object    | `{ mode: "first", maxAttachments: 1 }` | Selecteren welke overeenkomende bijlagen worden verwerkt             |
| `echoTranscript` | `boolean` | `false`                                | Alleen audio: het transcript weergeven vóór agentverwerking          |
| `echoFormat`     | `string`  | `'📝 "{transcript}"'`                  | Alleen audio: indeling voor het weergegeven transcript               |

Prompts, limieten, taalhints, aanvraagoverschrijvingen en provideropties kunnen als standaardwaarden voor mogelijkheden worden ingesteld of per afzonderlijke `tools.media.models[]`-vermelding worden overschreven. Standaardwaarden voor mogelijkheden gelden ook voor automatisch gedetecteerde providers wanneer geen expliciet model is geconfigureerd.

### Modelvermeldingen

Elke `models[]`-vermelding is een **provider**-vermelding (standaard) of een **CLI**-vermelding:

<Tabs>
  <Tab title="Providervermelding">
    ```json5
    {
      type: "provider", // standaard indien weggelaten
      provider: "openai",
      model: "gpt-5.6-sol",
      prompt: "Beschrijf de afbeelding in <= 500 tekens.",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"],
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  </Tab>
  <Tab title="CLI-vermelding">
    ```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "Lees de media op {{AttachmentPath}} en beschrijf deze in <= {{MaxChars}} tekens.",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    CLI-sjablonen kunnen ook `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{OutputDir}}` (werkmap gemaakt voor deze uitvoering) en `{{OutputBase}}` (basispad van werkbestand, zonder extensie) gebruiken. De oudere namen `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` en `{{MediaDir}}` blijven verouderde compatibiliteitsaliassen.

  </Tab>
</Tabs>

### Providerreferenties

Mediabegrip door providers gebruikt dezelfde authenticatieoplossing als normale modelaanroepen: authenticatieprofielen, omgevingsvariabelen en vervolgens `models.providers.<providerId>.apiKey`. `tools.media.models[]`-vermeldingen accepteren geen inline `apiKey`-veld.

```json5
{
  models: {
    providers: {
      openai: { apiKey: "<OPENAI_API_KEY>" },
      moonshot: { apiKey: "<MOONSHOT_API_KEY>" },
    },
  },
}
```

Zie [Tools en aangepaste providers](/nl/gateway/config-tools) voor profielen, omgevingsvariabelen en aangepaste basis-URL's.

## Regels en gedrag

- Media die groter zijn dan `maxBytes` slaan dat model over en proberen het volgende.
- Audiobestanden kleiner dan 1024 bytes worden als leeg/beschadigd behandeld en vóór transcriptie overgeslagen; de agent krijgt in plaats daarvan een deterministisch tijdelijk transcript.
- Als het actieve primaire afbeeldingsmodel standaard al visuele invoer ondersteunt, slaat OpenClaw het `[Image]`-samenvattingsblok over en geeft het de oorspronkelijke afbeelding rechtstreeks door aan het model. MiniMax is een uitzondering: `minimax`, `minimax-cn`, `minimax-portal` en `minimax-portal-cn` routeren afbeeldingsbegrip altijd via de door de plugin beheerde `MiniMax-VL-01`-mediaprovider, zelfs als verouderde MiniMax M2.x-chatmetagegevens beweren afbeeldingsinvoer te ondersteunen (alleen `MiniMax-M3` en hoger worden beschouwd als modellen met standaard visuele ondersteuning).
- Als een primair Gateway-/WebChat-model alleen tekst ondersteunt, blijven afbeeldingsbijlagen behouden als uitbestede `media://inbound/*`-referenties, zodat afbeeldings-/PDF-tools of een geconfigureerd afbeeldingsmodel ze nog steeds kunnen inspecteren in plaats van de bijlage kwijt te raken.
- Expliciete `openclaw infer image describe --file <path> --model <provider/model>` (alias: `openclaw capability image describe`) voert die provider/dat model met afbeeldingsondersteuning rechtstreeks uit, inclusief Ollama-referenties zoals `ollama/qwen2.5vl:7b` wanneer een overeenkomend model met afbeeldingsondersteuning is geconfigureerd onder `models.providers.ollama.models[]`.
- Als `<capability>.enabled` niet `false` is, maar er geen modellen zijn geconfigureerd, probeert OpenClaw het actieve antwoordmodel wanneer de provider daarvan de mogelijkheid ondersteunt.

### Automatische detectie (standaard)

Wanneer `tools.media.<capability>.enabled` niet `false` is en er geen modellen zijn geconfigureerd, probeert OpenClaw de volgende opties in deze volgorde en stopt het bij de eerste werkende optie:

<Steps>
  <Step title="Geconfigureerd afbeeldingsmodel (alleen afbeelding)">
    Primaire/terugvalreferenties van `agents.defaults.imageModel`, tenzij het actieve antwoordmodel standaard al visuele invoer ondersteunt. Geef voorkeur aan `provider/model`-referenties; kale referenties worden alleen gekwalificeerd op basis van geconfigureerde providermodelvermeldingen met afbeeldingsondersteuning wanneer de overeenkomst uniek is.
  </Step>
  <Step title="Actief antwoordmodel">
    Het actieve antwoordmodel, wanneer de provider daarvan de mogelijkheid ondersteunt.
  </Step>
  <Step title="Providerauthenticatie (alleen audio, vóór lokale CLI's)">
    Geconfigureerde `models.providers.*`-vermeldingen die audio ondersteunen, worden vóór lokale CLI's geprobeerd. Prioriteitsvolgorde van gebundelde providers (gelijke prioriteit wordt alfabetisch op provider-id beslist): Groq/OpenAI &rarr; xAI &rarr; Deepgram &rarr; OpenRouter &rarr; Google/SenseAudio &rarr; Deepinfra/ElevenLabs &rarr; Mistral.
  </Step>
  <Step title="Lokale CLI's (alleen audio)">
    Beschikbare lokale binaire bestanden vormen een geordende terugvallijst:
    - `whisper-cli` alleen als eerste nadat een eerdere modelaanroep in het huidige proces Metal of CUDA heeft waargenomen
    - Standaard voor CPU: `sherpa-onnx-offline` (vereist `SHERPA_ONNX_MODEL_DIR` met `tokens.txt`/`encoder.onnx`/`decoder.onnx`/`joiner.onnx`)
    - `whisper-cli` wanneer versnelling alleen bij het bouwen mogelijk is of niet is waargenomen
    - `parakeet-mlx` op Apple Silicon (geschikt voor MLX, apparaatgebruik niet waargenomen)
    - `whisper` (Python-CLI; gebruikt standaard het `turbo`-model en downloadt automatisch)

    Inspectie van backendmogelijkheden wordt gecachet en laadt geen model. Bouwmogelijkheden, aangevraagde backendvlaggen en de backend die tijdens een echte aanroep is waargenomen, blijven gescheiden. Automatisch gedetecteerde whisper.cpp laat logboekregistratie voor modeluitvoeringen ingeschakeld, zodat de door upstream geselecteerde backendregel kan worden vastgelegd. Expliciete CLI-vermeldingen behouden hun geconfigureerde volgorde, backendvlaggen en uitvoervlaggen.

  </Step>
  <Step title="Providerauthenticatie (afbeelding/video)">
    Geconfigureerde `models.providers.*`-vermeldingen die de mogelijkheid ondersteunen, worden vóór de gebundelde terugvalvolgorde geprobeerd. Configuratieproviders die alleen afbeeldingen ondersteunen en een model met afbeeldingsondersteuning hebben, registreren zich automatisch voor mediabegrip, zelfs wanneer ze geen gebundelde leveranciersplugin zijn.

    Prioriteitsvolgorde van gebundelde providers (gelijke prioriteit wordt alfabetisch op provider-id beslist):
    - Afbeelding: Anthropic/OpenAI &rarr; Google &rarr; MiniMax &rarr; Deepinfra &rarr; MiniMax Portal &rarr; Z.AI
    - Video: Google &rarr; Qwen &rarr; Moonshot

  </Step>
  <Step title="Antigravity-CLI (alleen afbeelding/video)">
    Het eerste geïnstalleerde binaire bestand `agy` of `antigravity` (overschrijven met `OPENCLAW_ANTIGRAVITY_CLI`), in een sandbox beperkt tot de map van de media.
  </Step>
</Steps>

Automatische detectie uitschakelen voor een mogelijkheid:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

<Note>
Detectie van binaire bestanden gebeurt naar beste vermogen op macOS/Linux/Windows; zorg dat de CLI in `PATH` staat (`~` wordt uitgevouwen), of stel een expliciete CLI-modelvermelding in met een volledig opdrachtpad.
</Note>

### Proxyondersteuning (provideroproepen voor audio/video)

Providergebaseerd begrip van **audio** en **video** respecteert standaardomgevingsvariabelen voor uitgaande proxy's, waaronder omzeilingsregels voor `NO_PROXY`/`no_proxy`: `HTTPS_PROXY`, `HTTP_PROXY`, `ALL_PROXY`, `https_proxy`, `http_proxy`, `all_proxy`. Variabelen in kleine letters hebben voorrang op variabelen in hoofdletters. Als geen van deze is ingesteld, gebruikt mediabegrip een directe uitgaande verbinding; als de proxywaarde onjuist is, registreert OpenClaw een waarschuwing en valt het terug op rechtstreeks ophalen. Afbeeldingsbegrip loopt niet via dit proxypad.

## Mogelijkheden

Stel `capabilities` in op een `models[]`-vermelding om deze te beperken tot specifieke mediatypen. Voor gedeelde lijsten leidt OpenClaw standaardwaarden af per gebundelde provider:

| Provider                                                                 | Mogelijkheden          |
| ------------------------------------------------------------------------ | --------------------- |
| `openai`, `anthropic`, `minimax`                                         | afbeelding                 |
| `minimax-portal`                                                         | afbeelding                 |
| `moonshot`                                                               | afbeelding + video         |
| `openrouter`                                                             | afbeelding + audio         |
| `google` (Gemini API)                                                    | afbeelding + audio + video |
| `qwen`                                                                   | afbeelding + video         |
| `deepinfra`                                                              | afbeelding + audio         |
| `mistral`                                                                | audio                 |
| `zai`                                                                    | afbeelding                 |
| `groq`, `xai`, `deepgram`, `senseaudio`                                  | audio                 |
| Elke `models.providers.<id>.models[]`-catalogus met een model dat afbeeldingen ondersteunt | afbeelding                 |

Stel voor CLI-vermeldingen `capabilities` expliciet in om onverwachte overeenkomsten te voorkomen; als dit wordt weggelaten, komt de vermelding in aanmerking voor elke mogelijkhedenlijst waarin deze voorkomt.

## Ondersteuningsmatrix voor providers

| Mogelijkheid | Providers                                                                                                                                               | Opmerkingen                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Afbeelding      | Anthropic, Codex app-server, Deepinfra, Google, MiniMax, MiniMax Portal, Moonshot, OpenAI, OpenAI Codex OAuth, OpenRouter, Qwen, Z.AI, configuratieproviders | Plugins van leveranciers registreren ondersteuning voor afbeeldingen; `openai/*` kan routering via een API-sleutel of Codex OAuth gebruiken; `codex/*` gebruikt een begrensde Codex app-server-beurt; configuratieproviders die afbeeldingen ondersteunen, worden automatisch geregistreerd. |
| Audio      | Deepgram, Deepinfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter, SenseAudio, xAI                                                             | Transcriptie door providers (Whisper/Groq/xAI/Deepgram/OpenRouter STT/Gemini/SenseAudio/Scribe/Voxtral).                                                                                     |
| Video      | Google, Moonshot, Qwen                                                                                                                                  | Videobegrip door providers via plugins van leveranciers; videobegrip van Qwen gebruikt de standaard DashScope-eindpunten.                                                                        |

<Note>
**Opmerking over MiniMax**: afbeeldingsbegrip voor `minimax`, `minimax-cn`, `minimax-portal` en `minimax-portal-cn` is altijd afkomstig van de door de plugin beheerde mediaprovider `MiniMax-VL-01`, zelfs als verouderde chatmetadata van MiniMax M2.x beweert dat afbeeldingsinvoer wordt ondersteund.
</Note>

## Richtlijnen voor modelselectie

- Geef voor elke mediamogelijkheid de voorkeur aan het krachtigste model van de huidige generatie wanneer kwaliteit en veiligheid belangrijk zijn.
- Vermijd oudere of zwakkere mediamodellen voor agents met ingeschakelde tools die niet-vertrouwde invoer verwerken.
- Houd voor beschikbaarheid ten minste één terugvaloptie per mogelijkheid aan (een kwaliteitsmodel en een sneller/goedkoper model).
- CLI-terugvalopties (`whisper-cli`, `whisper`, `gemini`) helpen wanneer provider-API's niet beschikbaar zijn.
- Bekende bestandsuitvoermodi zijn leidend: een leeg of ontbrekend afgeleid transcriptiebestand levert geen transcriptie op, in plaats van terug te vallen op CLI-voortgangsuitvoer.
- `parakeet-mlx`: gebruik `--output-format txt` (of `all`) met `--output-dir` en de standaarduitvoersjabloon `{filename}`. De upstream-omgevingsvariabelen `PARAKEET_OUTPUT_FORMAT` en `PARAKEET_OUTPUT_TEMPLATE` worden ook gerespecteerd. OpenClaw leest `<output-dir>/<media-basename>.txt`; de standaardindeling `srt`, andere indelingen en aangepaste uitvoersjablonen blijven stdout gebruiken.

## Beleid voor bijlagen

`attachments` per mogelijkheid bepaalt welke bijlagen worden verwerkt:

<ParamField path="mode" type='"first" | "all"' default="first">
  Verwerk alleen de eerste geselecteerde bijlage, of alle bijlagen.
</ParamField>
<ParamField path="maxAttachments" type="number" default="1">
  Beperk het aantal dat wordt verwerkt.
</ParamField>
<ParamField path="prefer" type='"first" | "last" | "path" | "url"'>
  Selectievoorkeur voor kandidaatbijlagen.
</ParamField>

Wanneer `mode: "all"`, krijgen uitvoerresultaten de labels `[Image 1/2]`, `[Audio 2/2]`, enzovoort.

### Extractie van bestandsbijlagen

- Geëxtraheerde bestandstekst wordt als niet-vertrouwde externe inhoud verpakt voordat deze aan de mediaprompt wordt toegevoegd, met grensmarkeringen zoals `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` en een metadataregel `Source: External`.
- Dit pad laat bewust de lange banner `SECURITY NOTICE:` weg om de mediaprompt kort te houden; de grensmarkeringen en metadata blijven van toepassing.
- Een bestand zonder extraheerbare tekst krijgt `[No extractable text]`.
- Als een PDF terugvalt op gerenderde pagina-afbeeldingen, stuurt OpenClaw die afbeeldingen door naar antwoordmodellen met beeldondersteuning en behoudt het de tijdelijke aanduiding `[PDF content rendered to images]` in het bestandsblok.

## Configuratievoorbeelden

<Tabs>
  <Tab title="Gedeelde modellen + overschrijvingen">
    ```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.6-sol", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "Lees de media op {{AttachmentPath}} en beschrijf deze in <= {{MaxChars}} tekens.",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Alleen audio + video">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{AttachmentPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Lees de media op {{AttachmentPath}} en beschrijf deze in <= {{MaxChars}} tekens.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Alleen afbeeldingen">
    ```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.6-sol" },
              { provider: "anthropic", model: "claude-opus-5" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Lees de media op {{AttachmentPath}} en beschrijf deze in <= {{MaxChars}} tekens.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Eén multimodale vermelding">
    ```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Statusuitvoer

Wanneer mediabegrip wordt uitgevoerd, bevat `/status` een samenvattingsregel per mogelijkheid:

```
📎 Media: afbeelding geslaagd (openai/gpt-5.6-sol) · audio geslaagd (whisper-cli waargenomen=metal)
```

Voer voor de preflight-inventarisatie `openclaw capability audio providers` uit. Lokale rijen tonen de lokale winnaar van de terugvalopties afzonderlijk van de algemene providerselectie, gereedheid en afzonderlijke backendvelden voor geschikt/aangevraagd/waargenomen. Dezelfde lokale selectie is beschikbaar als informatieve doctor-bevinding:

```bash
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

## Opmerkingen

- Begrip wordt naar beste vermogen uitgevoerd. Fouten blokkeren antwoorden niet.
- Bijlagen worden nog steeds aan modellen doorgegeven wanneer begrip is uitgeschakeld.
- Gebruik `scope` om te beperken waar begrip wordt uitgevoerd (bijvoorbeeld alleen in DM's).

## Gerelateerd

- [Configuratie](/nl/gateway/configuration)
- [Ondersteuning voor afbeeldingen en media](/nl/nodes/images)
