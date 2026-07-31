---
read_when:
    - Talk-modus implementeren op macOS/iOS/Android
    - Spraak-/TTS-/onderbrekingsgedrag wijzigen
summary: 'Praatmodus: doorlopende spraakgesprekken via lokale STT/TTS en realtime spraakcommunicatie'
title: Spreekmodus
x-i18n:
    generated_at: "2026-07-27T05:10:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b21319eee169ba898331f87279a2b2a5170441131a1e9cdc85c15b268d165e21
    source_path: nodes/talk.md
    workflow: 16
---

Talk-modus omvat vijf runtimevormen:

- **Native Talk op macOS/iOS/Android**: native spraakherkenning, chat via de Gateway en `talk.speak`-TTS. Apple-spraakherkenning op macOS/iOS kan netwerkdiensten gebruiken; het gedrag op Android is afhankelijk van de geïnstalleerde spraakdienst. Nodes maken de `talk`-mogelijkheid bekend en geven aan welke `talk.*`-opdrachten ze ondersteunen.
- **iOS Talk (realtime)**: WebRTC onder beheer van de client voor realtimeconfiguraties van OpenAI die `webrtc` als transport selecteren of geen transport opgeven. Expliciete `gateway-relay`-, `provider-websocket`- en niet-OpenAI-realtimeconfiguraties blijven op de relay onder beheer van de Gateway; niet-realtimeconfiguraties gebruiken de native spraaklus.
- **Browser Talk**: `talk.client.create` voor `webrtc`-/`provider-websocket`-sessies onder beheer van de client, of `talk.session.create` voor `gateway-relay`-sessies onder beheer van de Gateway. `managed-room` is gereserveerd voor overdracht aan de Gateway en portofoonruimten.
- **Android Talk (realtime)**: schakel dit in met `talk.realtime.mode: "realtime"` en `talk.realtime.transport: "gateway-relay"`. Anders blijft Android native spraakherkenning, chat via de Gateway en `talk.speak` gebruiken.
- **Clients uitsluitend voor transcriptie**: `talk.session.create({ mode: "transcription", transport: "gateway-relay", brain: "none" })`, gevolgd door `talk.session.appendAudio`, `talk.session.cancelTurn` en `talk.session.close` voor ondertiteling/dicteren zonder gesproken antwoord van een assistent. Eenmalig geüploade spraaknotities gebruiken nog steeds het audiopad voor [mediabegrip](/nl/nodes/media-understanding).

Native Talk is een continue lus: luisteren naar spraak, het transcript via de actieve sessie naar het model sturen, op het antwoord wachten en het vervolgens uitspreken via de geconfigureerde Talk-provider (`talk.speak`).

Realtime Talk onder beheer van de client stuurt toolaanroepen van de provider door via `talk.client.toolCall` in plaats van `chat.send` rechtstreeks aan te roepen. Terwijl een realtimeconsult actief is, kunnen clients `talk.client.steer` of `talk.session.steer` aanroepen om gesproken invoer te classificeren als `status`, `steer`, `cancel` of `followup`. Geaccepteerde bijsturing wordt in de actieve ingebedde uitvoering in de wachtrij geplaatst; afgewezen bijsturing retourneert een reden zoals `no_active_run`, `not_streaming` of `compacting`.

Definitief vastgelegde realtime-uitingen van de gebruiker en assistent worden altijd direct aan de actieve agentsessie toegevoegd, zodat latere chat- en spraakbeurten één geschiedenis delen. Transports onder beheer van de client rapporteren hun definitieve transcripties met stabiele invoer-id's; relaysessies van de Gateway voegen dezelfde gebeurtenissen aan de serverzijde toe. Providersessies ontvangen ook de begrensde realtimeprofielcontext die door Discord-spraak wordt gebruikt.

Voor consultuitvoeringen die via spraak zijn gestart, is een nieuwe, exacte gesproken bevestiging vereist vóór ingrijpende acties, zoals berichten verzenden, Nodes bedienen, browser-/computeracties, servicewijzigingen, destructieve shellopdrachten of publicatie. De bevestiging geldt alleen voor de exacte argumenten van de geblokkeerde tool en wordt eenmaal verbruikt; niet-gerelateerde gelijktijdige uitvoeringen blijven onaangetast. Wanneer een gesprek wordt beëindigd, kan OpenClaw een compacte samenvatting **Wijzigingen door spraakoproep** voor muterende tools verzenden naar het laatste niet-WebChat-afleverdoel van de sessie.

Talk uitsluitend voor transcriptie verzendt dezelfde Talk-gebeurtenisenvelop als realtime- en STT/TTS-sessies, maar gebruikt `mode: "transcription"` en `brain: "none"`. Alle Talk-sessies zenden gebeurtenissen uit via het `talk.event`-kanaal; clients abonneren zich hierop voor gedeeltelijke/definitieve transcriptupdates (`transcript.delta`/`transcript.done`) en andere sessietelemetrie.

Browser Video Talk is beschikbaar voor OpenAI Realtime WebRTC- en Google Live-
providersessies via WebSocket. OpenAI ontvangt één begrensde JPEG wanneer
`describe_view` om visuele context vraagt; er wordt geen continue
camerastream ontvangen. Google Live ontvangt begrensde JPEG-frames rechtstreeks vanuit de
browser met maximaal één frame per seconde, terwijl `describe_view` de
status van de camerastream rapporteert. In beide gevallen omzeilen cameraframes de Gateway en
worden de camera- en microfoontracks vrijgegeven wanneer Talk wordt gestopt.

## Gedrag (macOS)

- Altijd zichtbare overlay wanneer de Talk-modus is ingeschakeld.
- Faseovergangen **Luisteren &rarr; Denken &rarr; Spreken**.
- Na een korte pauze (stiltevenster) wordt het huidige transcript verzonden.
- Antwoorden worden naar WebChat geschreven (hetzelfde als typen).
- **Onderbreken bij spraak** (standaard ingeschakeld): als de gebruiker praat terwijl de assistent spreekt, stopt het afspelen en wordt het tijdstip van de onderbreking vastgelegd voor de volgende prompt.

## Spraakrichtlijnen in antwoorden

De assistent kan een antwoord vooraf laten gaan door één JSON-regel om de stem te beheren:

```json
{ "voice": "<voice-id>", "once": true }
```

Regels:

- Alleen de eerste niet-lege regel; de JSON-regel wordt vóór het afspelen via TTS verwijderd.
- Onbekende sleutels worden genegeerd.
- `once: true` is alleen van toepassing op het huidige antwoord; zonder deze sleutel wordt de stem de nieuwe standaard voor de Talk-modus.

Ondersteunde sleutels: `voice` / `voice_id` / `voiceId`, `model` / `model_id` / `modelId`, `speed`, `rate` (woorden per minuut), `stability`, `similarity`, `style`, `speakerBoost`, `seed`, `normalize`, `lang`, `output_format`, `latency_tier`, `once`.

## Configuratie (`~/.openclaw/openclaw.json`)

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          apiKey: "openai_api_key",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Speak warmly and keep answers brief.",
      mode: "realtime",
      transport: "webrtc",
      brain: "agent-consult",
    },
  },
}
```

| Sleutel                                  | Standaard                                  | Opmerkingen                                                                                                                                                                                                                                                                 |
| ---------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`                               | -                                          | TTS-provider voor Active Talk. Gebruik `elevenlabs`, `mlx` of `system` voor lokale afspeelpaden op macOS.                                                                                                                                                                             |
| `providers.<id>.voiceId`                 | -                                          | ElevenLabs valt terug op `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID`, of op de eerste beschikbare stem met een API-sleutel.                                                                                                                                                             |
| `speechLocale`                           | apparaatstandaard                          | BCP 47-landinstelling voor systeemeigen spraakherkenning op Android, iOS en macOS. Apple Speech kan netwerkdiensten gebruiken; Android stuurt ook de taalcomponent door naar realtime invoertranscriptie.                                                                                  |
| `providers.elevenlabs.modelId`           | `eleven_v3`                                |                                                                                                                                                                                                                                                                            |
| `providers.mlx.modelId`                  | `mlx-community/Soprano-80M-bf16`           |                                                                                                                                                                                                                                                                            |
| `providers.elevenlabs.apiKey`            | -                                          | Valt terug op `ELEVENLABS_API_KEY` (of het shellprofiel van de Gateway indien beschikbaar).                                                                                                                                                                                                |
| `silenceTimeoutMs`                       | `700` ms macOS/Android, `900` ms iOS       | Pauzevenster voordat Talk het transcript verzendt.                                                                                                                                                                                                                             |
| `interruptOnSpeech`                      | `true`                                     |                                                                                                                                                                                                                                                                            |
| `outputFormat`                           | `pcm_44100` macOS/iOS, `pcm_24000` Android | Stel `mp3_*` in om MP3-streaming af te dwingen.                                                                                                                                                                                                                                        |
| `consultThinkingLevel`                   | niet ingesteld                            | Overschrijving van het denkniveau voor de agentuitvoering achter realtime `openclaw_agent_consult`-aanroepen.                                                                                                                                                                                  |
| `consultFastMode`                        | niet ingesteld                            | Overschrijving van de snelle modus voor realtime `openclaw_agent_consult`-aanroepen.                                                                                                                                                                                                            |
| `realtime.provider`                      | -                                          | `openai` voor WebRTC, `google` voor de WebSocket van de provider, of een provider die alleen via een bridge werkt via het Gateway-relais.                                                                                                                                                                     |
| `realtime.providers.<id>`                | -                                          | Realtimeconfiguratie die eigendom is van de provider. Browsers ontvangen alleen tijdelijke/beperkte sessiereferenties, nooit een standaard-API-sleutel.                                                                                                                                                 |
| `realtime.providers.openai.speakerVoice` | `alloy`                                    | Ingebouwde stem-id van OpenAI Realtime (de oudere sleutel `voice` werkt nog, maar is verouderd). Huidige `gpt-realtime-2.1`-stemmen: `alloy`, `ash`, `ballad`, `cedar`, `coral`, `echo`, `marin`, `sage`, `shimmer`, `verse`; `marin` en `cedar` worden aanbevolen voor de beste kwaliteit. |
| `realtime.transport`                     | -                                          | `webrtc`: door de client beheerde OpenAI WebRTC op iOS en in de browser. `provider-websocket`: door de browser beheerd, blijft op iOS op het Gateway-relais. `gateway-relay`: houdt provideraudio op de Gateway; Android gebruikt realtime alleen met dit transport.                                  |
| `realtime.brain`                         | -                                          | `agent-consult` routeert realtime toolaanroepen via het Gateway-beleid; `direct-tools` biedt verouderde compatibiliteit voor directe tools; `none` is bedoeld voor transcriptie/externe orkestratie.                                                                                                 |
| `realtime.consultRouting`                | -                                          | `provider-direct` behoudt het directe antwoord van de provider wanneer deze `openclaw_agent_consult` overslaat; `force-agent-consult` routeert afgeronde gebruikerstranscripten in plaats daarvan via OpenClaw.                                                                                          |
| `realtime.instructions`                  | -                                          | Voegt systeeminstructies voor de provider toe aan de ingebouwde realtimeprompt van OpenClaw (stemstijl/toon); de standaardrichtlijnen van `openclaw_agent_consult` blijven behouden.                                                                                                                |

`talk.catalog` maakt canonieke provider-id's en registeraliassen beschikbaar, evenals de geldige modi, transporten, brain-strategieën, realtime audio-indelingen en capaciteitsvlaggen van elke provider, plus het tijdens runtime geselecteerde gereedheidsresultaat. Eigen Talk-clients moeten die catalogus lezen in plaats van provideraliassen lokaal bij te houden; beschouw een oudere Gateway die groepsgereedheid weglaat als niet-geverifieerd in plaats van definitief niet-geconfigureerd. Providers voor streamingtranscriptie worden ontdekt via `talk.catalog.transcription`; het huidige Gateway-relais gebruikt de configuratie van de Voice Call-streamingprovider totdat een specifiek configuratieoppervlak voor Talk-transcriptie wordt uitgebracht.

## macOS-interface

- Schakelaar in de menubalk: **Talk**
- Configuratietabblad: groep **Talk Mode** (stem-id + onderbrekingsschakelaar)
- Overlay: de bol geeft de universele Talk-golfvorm weer (gedeeld met iOS, watchOS en Android). Tijdens luisteren volgt deze het live microfoonniveau, tijdens spreken de daadwerkelijke afspeelenvelop van TTS en tijdens denken ademt deze zacht. Klik op de bol om te pauzeren/hervatten, dubbelklik om het spreken te stoppen en klik op X om de Talk-modus te verlaten.

## Android-interface

- De hoofdnavigatie van Android bestaat uit **Home**, **Chat** en **Settings**. Spraakinvoer
  bevindt zich in het invoerveld van Chat in plaats van op een afzonderlijk tabblad Voice.
- Tik op de microfoon van het invoerveld voor dicteren op het apparaat. Houd deze ingedrukt om
  een spraaknotitiebijlage op te nemen. Start doorlopende Talk via de Talk-golfvorm.
- Dicteren, het opnemen van spraaknotities en Talk zijn elkaar uitsluitende microfoonpaden;
  wanneer je er één start, worden de andere gestopt of geblokkeerd.
- Realtime Talk geeft de voorkeur aan de microfoon van een verbonden Bluetooth Classic- of BLE-headset;
  als de verbinding wordt verbroken, vraagt de app om een andere headsetinvoer of
  valt deze terug op de standaardmicrofoon, waarbij de standaardvoorkeur wordt hersteld zodra
  de opname stopt.
- Dicteren en het opnemen van spraaknotities stoppen wanneer de app de voorgrond verlaat of
  de gebruiker Chat verlaat.
- Talk Mode blijft actief totdat deze wordt uitgeschakeld of de Node de verbinding verbreekt, waarbij tijdens activiteit het Android-type voorgrondservice voor de microfoon wordt gebruikt.
- Android ondersteunt de uitvoerindelingen `pcm_16000`, `pcm_22050`, `pcm_24000` en `pcm_44100` voor `AudioTrack`-streaming met lage latentie.

## Opmerkingen

- Vereist machtigingen voor spraak en microfoon.
- Systeemeigen Talk gebruikt de actieve Gateway-sessie en valt alleen terug op het pollen van de geschiedenis wanneer antwoordgebeurtenissen niet beschikbaar zijn.
- De Gateway verwerkt Talk-weergave via `talk.speak` met de actieve Talk-provider. Android valt alleen terug op lokale systeem-TTS wanneer die RPC niet beschikbaar is.
- Lokale MLX-weergave op macOS gebruikt de gebundelde helper `openclaw-mlx-tts` wanneer deze aanwezig is, of een uitvoerbaar bestand op `PATH`. Stel `OPENCLAW_MLX_TTS_BIN` in om tijdens de ontwikkeling naar een aangepast uitvoerbaar helperbestand te verwijzen.
- Waardebereiken voor stemdirectieven (ElevenLabs): `stability`, `similarity` en `style` accepteren `0..1`; `speed` accepteert `0.5..2`; `latency_tier` accepteert `0..4`.

## Gerelateerd

- [Spraakactivering](/nl/nodes/voicewake)
- [Audio en spraaknotities](/nl/nodes/audio)
- [Mediabegrip](/nl/nodes/media-understanding)
