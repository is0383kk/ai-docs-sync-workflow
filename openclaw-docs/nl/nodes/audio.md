---
read_when:
    - Audiotranscriptie of mediaverwerking wijzigen
summary: Hoe inkomende audio-/spraakberichten worden gedownload, getranscribeerd en in antwoorden worden ingevoegd
title: Audio- en spraakberichten
x-i18n:
    generated_at: "2026-07-27T05:04:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4076e3e55eb5c6dcc94cfdd842619697c8d756b924956d7b266d18446b4dd9be
    source_path: nodes/audio.md
    workflow: 16
---

## Wat het doet

Wanneer audiobegrip is ingeschakeld (of automatisch wordt gedetecteerd), doet OpenClaw het volgende:

1. Zoekt de eerste audiobijlage (lokaal pad of URL) en downloadt deze indien nodig.
2. Handhaaft `maxBytes` voordat de bijlage naar elk modelitem wordt verzonden.
3. Voert het eerste geschikte modelitem op volgorde uit (provider of CLI); als een item mislukt of wordt overgeslagen (grootte/time-out), wordt het volgende item geprobeerd.
4. Vervangt bij succes `Body` door een `[Audio]`-blok en stelt `{{Transcript}}` in.

Wanneer de transcriptie slaagt, worden `CommandBody`/`RawBody` ook op het transcript ingesteld, zodat slash-opdrachten blijven werken. Met `--verbose` tonen de logboeken wanneer de transcriptie wordt uitgevoerd en wanneer deze de berichttekst vervangt.

## Automatische detectie (standaard)

Als je geen modellen hebt geconfigureerd en `tools.media.audio.enabled` niet `false` is, detecteert OpenClaw automatisch in deze volgorde en stopt het bij de eerste werkende optie:

1. **Actief antwoordmodel**, wanneer de provider daarvan audiobegrip ondersteunt.
2. **Geconfigureerde providerauthenticatie** — elk `models.providers.*`-item waarvoor authenticatie beschikbaar is voor een provider die audiotranscriptie ondersteunt. Dit wordt vóór lokale CLI's gecontroleerd, zodat een geconfigureerde API-sleutel altijd voorrang krijgt op een lokaal binair bestand in `PATH`.
   Providerprioriteit wanneer er meerdere zijn geconfigureerd: Groq, OpenAI, xAI, Deepgram, Google, SenseAudio, ElevenLabs, Mistral.
3. **Lokale CLI's** (alleen als geen providerauthenticatie is gevonden). OpenClaw stelt een geordende lijst met terugvalopties samen:
   - `whisper-cli`, vóór CPU-standaardopties, maar alleen wanneer een eerdere modelaanroep in het huidige proces Metal of CUDA heeft waargenomen
   - `sherpa-onnx-offline` op de standaard-CPU-provider (vereist `SHERPA_ONNX_MODEL_DIR` met `tokens.txt`, `encoder.onnx`, `decoder.onnx` en `joiner.onnx`)
   - `whisper-cli` wanneer Metal/CUDA alleen bij het bouwen beschikbaar is of de geselecteerde backend anderszins niet is waargenomen
   - `parakeet-mlx` op Apple Silicon (geschikt voor MLX; apparaatgebruik blijft niet-waargenomen)
   - `whisper` (Python-CLI; downloadt modellen automatisch)

Herkomst van installatie/koppeling is bewijs van capaciteit, niet van uitvoering. Hierdoor wordt een kandidaat nooit op zichzelf vóór CPU-sherpa geplaatst. OpenClaw laadt tijdens installatie- of statuscontroles geen model alleen om een backend te testen.
Automatisch gedetecteerde whisper.cpp houdt de normale logboekregistratie voor modeluitvoering ingeschakeld, zodat OpenClaw de bovenliggende `using … backend`-regel kan vastleggen. Expliciete CLI-items behouden hun geconfigureerde uitvoervlaggen.

Automatische detectie van de Gemini CLI voor mediabegrip is vervangen door een gesandboxte Antigravity CLI-terugvaloptie (`agy`) voor afbeeldingen/video; audio gebruikt naast de bovenstaande lokale binaire bestanden geen CLI-terugvaloptie.

Stel `tools.media.audio.enabled: false` in om automatische detectie uit te schakelen. Voeg items met capaciteitstags toe aan `tools.media.models` om dit aan te passen.

<Note>
Detectie van binaire bestanden gebeurt op basis van redelijke inspanning voor macOS/Linux/Windows. Zorg dat de CLI in `PATH` staat (`~` wordt uitgebreid), of stel een expliciet CLI-model met een volledig opdrachtpad in.
</Note>

Inspecteer de lokale selectie zonder audio te transcriberen:

```bash
openclaw capability audio providers
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

De providerinventaris rapporteert de winnaar van de lokale terugvalopties afzonderlijk van de globale providerselectie, plus velden voor beschikbare, aangevraagde en waargenomen backends. Nadat transcriptie is uitgevoerd, rapporteert `/status` de aangevraagde of waargenomen backend in de mediaregel. Expliciete audiogeschikte `tools.media.models`-CLI-items omzeilen nog steeds de automatische selectie; gebruik hun backendspecifieke vlaggen, zoals sherpa `--provider=cuda` of whisper.cpp `--no-gpu`/`--device`.

## Configuratievoorbeelden

### Provider + CLI-terugvaloptie (OpenAI + Whisper CLI)

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-4o-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          timeoutSeconds: 45,
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-transcribe" },
    },
  },
}
```

### Alleen provider (Deepgram)

```json5
{
  tools: {
    media: {
      models: [{ provider: "deepgram", model: "nova-3", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Alleen provider (Mistral Voxtral)

```json5
{
  tools: {
    media: {
      models: [{ provider: "mistral", model: "voxtral-mini-latest", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Alleen provider (SenseAudio)

```json5
{
  tools: {
    media: {
      models: [
        {
          provider: "senseaudio",
          model: "senseaudio-asr-pro-1.5-260319",
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true },
    },
  },
}
```

### Transcript naar de chat terugsturen (opt-in)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
    },
  },
}
```

## Opmerkingen en limieten

- Providerauthenticatie volgt de standaardvolgorde voor modelauthenticatie (authenticatieprofielen, omgevingsvariabelen, `models.providers.*.apiKey`).
- Installatiedetails voor Groq: [Groq](/nl/providers/groq).
- Deepgram neemt `DEEPGRAM_API_KEY` over wanneer `provider: "deepgram"` wordt gebruikt. Installatiedetails: [Deepgram](/nl/providers/deepgram).
- Installatiedetails voor Mistral: [Mistral](/nl/providers/mistral).
- SenseAudio neemt `SENSEAUDIO_API_KEY` over wanneer `provider: "senseaudio"` wordt gebruikt. Installatiedetails: [SenseAudio](/nl/providers/senseaudio).
- Audioproviders kunnen standaardwaarden onder `tools.media.audio` gebruiken of `baseUrl`, `headers`, `providerOptions` en limieten in hun `tools.media.models[]`-item overschrijven.
- De ingebouwde maximale audiogrootte is 20MB. Een overschrijving via `maxBytes` op itemniveau kan dit wijzigen; te grote audio wordt voor dat model overgeslagen en het volgende item wordt geprobeerd.
- Audiobestanden kleiner dan 1024 bytes worden vóór transcriptie door een provider/CLI overgeslagen.
- De standaardwaarde van `maxChars` voor audio is **niet ingesteld** (volledig transcript). Stel `tools.media.audio.maxChars` of `maxChars` per item in om de uitvoer in te korten.
- De standaardwaarde voor automatische detectie van OpenAI is `gpt-4o-transcribe`; stel `model: "gpt-4o-mini-transcribe"` in voor een goedkopere/snellere optie.
- Het transcript is voor sjablonen beschikbaar als `{{Transcript}}`.
- `tools.media.audio.echoTranscript` is standaard uitgeschakeld; `echoFormat` accepteert een tijdelijke aanduiding `{transcript}`.
- De stdout van de CLI is beperkt tot 5MB; houd CLI-uitvoer beknopt.
- CLI-`args` moet `{{AttachmentPath}}` gebruiken voor het pad naar het lokale audiobestand. Voer `openclaw doctor --fix` uit om verouderde tijdelijke aanduidingen `{input}` uit oudere `audio.transcription.command`-configuraties te migreren (ingetrokken sleutel: `audio.transcription`, vervangen door `tools.media.models`). `{{MediaPath}}` blijft een verouderde compatibiliteitsalias.
- `tools.media.concurrency` begrenst mediataken; het is geen GPU-planner.

### Permanente lokale STT

Automatisch gedetecteerde lokale STT blijft een afzonderlijk proces per aanvraag gebruiken. OpenClaw beheert momenteel geen permanente whisper.cpp-server, omdat het standaardpakket `whisper-cpp` van Homebrew die server uitschakelt, terwijl het bovenliggende voorbeeld geen geconfigureerde begrensde toelatingswachtrij heeft. Een door een Plugin beheerde permanente levenscyclus vereist een onderhouden, verpakte worker met status-/opstartcontrole, modelresidentie, begrensde wachtrijen, annulering/time-out, werking zonder authenticatie uitsluitend via loopback en geen cloudterugval voordat deze veilig kan worden ingeschakeld.

### Ondersteuning voor proxyomgevingen

Providergebaseerde audiotranscriptie respecteert standaardomgevingsvariabelen voor uitgaande proxy's, overeenkomstig de `EnvHttpProxyAgent`-semantiek van undici:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `ALL_PROXY` / `all_proxy`

Variabelen in kleine letters hebben voorrang op variabelen in hoofdletters; `NO_PROXY`/`no_proxy`-items (hostnamen, `*.suffix` of `host:port`) omzeilen de proxy. Als er geen proxyomgevingsvariabelen zijn ingesteld, wordt een directe uitgaande verbinding gebruikt. Als het instellen van de proxy mislukt (ongeldige URL), registreert OpenClaw een waarschuwing en valt het terug op rechtstreeks ophalen.

## Vermeldingsdetectie in groepen

Op kanalen die audiovoorcontrole ondersteunen, transcribeert OpenClaw audio **voordat** het op vermeldingen controleert wanneer `requireMention: true` voor een groepschat is ingesteld. Hierdoor kan een spraakbericht zonder bijschrift door de vermeldingspoort komen wanneer het transcript een geconfigureerd vermeldingspatroon bevat. Kanaalspecifieke documentatie beschrijft transportsystemen waarvoor een getypte vermelding vereist is.

**Zo werkt het:**

1. Als een spraakbericht geen tekst bevat en de groep vermeldingen vereist, voert OpenClaw een voorcontroletranscriptie van de eerste audiobijlage uit.
2. Het transcript wordt gecontroleerd op vermeldingspatronen (bijvoorbeeld `@BotName`, emoji-triggers).
3. Als een vermelding wordt gevonden, gaat het bericht door naar de volledige antwoordpijplijn.

**Terugvalgedrag:** als de voorcontroletranscriptie mislukt (time-out, API-fout enzovoort), valt het bericht terug op vermeldingsdetectie op basis van alleen tekst, zodat gemengde berichten (tekst + audio) nooit worden verwijderd.

**Afmelden per Telegram-groep/-onderwerp:**

- Stel `channels.telegram.groups.<chatId>.disableAudioPreflight: true` in om vermeldingscontroles via voorcontroletranscriptie voor die groep over te slaan.
- Stel `channels.telegram.groups.<chatId>.topics.<threadId>.disableAudioPreflight` in om dit per onderwerp te overschrijven (`true` om over te slaan, `false` om geforceerd in te schakelen).
- De standaardwaarde is `false` (voorcontrole ingeschakeld wanneer aan de voorwaarden voor verplichte vermeldingen is voldaan).

**Voorbeeld:** een gebruiker stuurt in een Telegram-groep met `requireMention: true` een spraakbericht met de tekst "Hé @Claude, wat voor weer is het?". Het spraakbericht wordt getranscribeerd, de vermelding wordt gedetecteerd en de agent antwoordt.

## Aandachtspunten

- Bereikregels gebruiken de eerste overeenkomst; `chatType` wordt genormaliseerd naar `direct`, `group` of `channel`.
- Zorg dat je CLI afsluit met code 0 en platte tekst afdrukt; JSON-uitvoer moet via `jq -r .text` worden aangepast.
- Bekende bestandsuitvoermodi zijn leidend: een leeg of ontbrekend afgeleid transcriptbestand levert geen transcript op, in plaats van terug te vallen op voortgangsuitvoer van de CLI.
- Gebruik voor `parakeet-mlx` `--output-format txt` (of `all`) met `--output-dir` en de standaarduitvoersjabloon `{filename}`. De bovenliggende omgevingsvariabelen `PARAKEET_OUTPUT_FORMAT` en `PARAKEET_OUTPUT_TEMPLATE` worden ook gerespecteerd. OpenClaw leest `<output-dir>/<media-basename>.txt`; de standaardindeling `srt`, andere indelingen en aangepaste uitvoersjablonen blijven stdout gebruiken.
- Houd time-outs redelijk (`timeoutSeconds`, standaard 60s) om te voorkomen dat de antwoordwachtrij wordt geblokkeerd.
- Voorcontroletranscriptie verwerkt alleen de **eerste** audiobijlage voor vermeldingsdetectie. Aanvullende audiobijlagen worden tijdens de hoofdfase voor mediabegrip verwerkt.

## Gerelateerd

- [Mediabegrip](/nl/nodes/media-understanding)
- [Praatmodus](/nl/nodes/talk)
- [Spraakactivering](/nl/nodes/voicewake)
