---
read_when:
    - Op zoek naar een overzicht van de mediamogelijkheden van OpenClaw
    - Bepalen welke mediaprovider je configureert
    - Begrijpen hoe asynchrone mediageneratie werkt
sidebarTitle: Media overview
summary: Mogelijkheden voor beeld, video, muziek, spraak en mediabegrip in één oogopslag
title: Mediaoverzicht
x-i18n:
    generated_at: "2026-07-27T06:36:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 18eb79e6915c5dc8d705bf5cadfcdddecaf7d21a037f102696d4f2bcd41e5bea
    source_path: tools/media-overview.md
    workflow: 16
---

OpenClaw genereert afbeeldingen, video's en muziek, begrijpt inkomende media
(afbeeldingen, audio, video) en spreekt antwoorden hardop uit met tekst-naar-spraak. Alle
mediamogelijkheden worden via tools aangestuurd: de agent bepaalt op basis
van het gesprek wanneer deze worden gebruikt, en elke tool verschijnt alleen wanneer
ten minste één ondersteunende provider is geconfigureerd.

Live spraak gebruikt het Talk-sessiecontract in plaats van het pad voor eenmalige mediatools.
Talk heeft drie modi: provider-native `realtime`, lokale of streaming
`stt-tts` en `transcription` voor spraakopname uitsluitend ter observatie. Deze modi
delen providercatalogi, event-enveloppen en annuleringssemantiek met
telefonie, vergaderingen, browser-realtime en native push-to-talk-clients.

## Mogelijkheden

<CardGroup cols={2}>
  <Card title="Afbeeldingen genereren" href="/nl/tools/image-generation" icon="image">
    Maak en bewerk afbeeldingen op basis van tekstprompts of referentieafbeeldingen via
    `image_generate`. Asynchroon in chatsessies — wordt op de achtergrond uitgevoerd en
    plaatst het resultaat zodra het gereed is.
  </Card>
  <Card title="Video's genereren" href="/nl/tools/video-generation" icon="video">
    Tekst-naar-video, afbeelding-naar-video en video-naar-video via `video_generate`.
    Asynchroon — wordt op de achtergrond uitgevoerd en plaatst het resultaat zodra het gereed is.
  </Card>
  <Card title="Muziek genereren" href="/nl/tools/music-generation" icon="music">
    Genereer muziek of audiotracks via `music_generate`. Asynchroon in
    chatsessies binnen de gedeelde taaklevenscyclus voor mediageneratie.
  </Card>
  <Card title="Tekst-naar-spraak" href="/nl/tools/tts" icon="microphone">
    Zet uitgaande antwoorden om in gesproken audio via de tool `tts` plus
    de configuratie `tts`. Synchroon.
  </Card>
  <Card title="Mediabegrip" href="/nl/nodes/media-understanding" icon="eye">
    Vat inkomende afbeeldingen, audio en video samen met modelproviders
    die visuele invoer ondersteunen en gespecialiseerde plugins voor mediabegrip.
  </Card>
  <Card title="Spraak-naar-tekst" href="/nl/nodes/audio" icon="ear-listen">
    Transcribeer inkomende spraakberichten via batch-STT of streaming-STT-providers
    voor Voice Call.
  </Card>
</CardGroup>

## Matrix met providermogelijkheden

<Note>
Deze tabel behandelt de gespecialiseerde plugins voor mediageneratie, TTS en STT. Veel
chatmodelproviders (Anthropic, Google, OpenAI en andere) begrijpen ook
inkomende media via hun antwoordmodel; bekijk de volledige providerlijst in
[Mediabegrip](/nl/nodes/media-understanding#provider-support-matrix).
</Note>

| Provider          | Afbeelding | Video | Muziek | TTS | STT | Realtime spraak | Mediabegrip |
| ----------------- | :--------: | :---: | :----: | :-: | :-: | :-------------: | :---------: |
| Alibaba           |            |   ✓   |        |     |     |                 |             |
| Azure Speech      |            |       |        |  ✓  |     |                 |             |
| BytePlus          |            |   ✓   |        |     |     |                 |             |
| ComfyUI           |     ✓      |   ✓   |   ✓    |     |     |                 |             |
| Deepgram          |            |       |        |     |  ✓  |                 |             |
| DeepInfra         |     ✓      |   ✓   |        |  ✓  |  ✓  |                 |      ✓      |
| ElevenLabs        |            |       |        |  ✓  |  ✓  |                 |             |
| fal               |     ✓      |   ✓   |   ✓    |     |     |                 |             |
| Google            |     ✓      |   ✓   |   ✓    |  ✓  |  ✓  |        ✓        |      ✓      |
| Gradium           |            |       |        |  ✓  |     |                 |             |
| Inworld           |            |       |        |  ✓  |     |                 |             |
| LiteLLM           |     ✓      |       |        |     |     |                 |             |
| Lokale CLI        |            |       |        |  ✓  |     |                 |             |
| Microsoft         |            |       |        |  ✓  |     |                 |             |
| Microsoft Foundry |     ✓      |       |        |     |     |                 |             |
| MiniMax           |     ✓      |   ✓   |   ✓    |  ✓  |     |                 |             |
| Mistral           |            |       |        |     |  ✓  |                 |             |
| OpenAI            |     ✓      |   ✓   |        |  ✓  |  ✓  |        ✓        |      ✓      |
| OpenRouter        |     ✓      |   ✓   |   ✓    |  ✓  |  ✓  |                 |      ✓      |
| PixVerse          |            |   ✓   |        |     |     |                 |             |
| Qwen              |            |   ✓   |        |     |     |                 |      ✓      |
| Runway            |            |   ✓   |        |     |     |                 |             |
| SenseAudio        |            |       |        |     |  ✓  |                 |             |
| Together          |            |   ✓   |        |     |     |                 |             |
| Volcengine        |            |       |        |  ✓  |     |                 |             |
| Vydra             |     ✓      |   ✓   |        |  ✓  |     |                 |             |
| xAI               |     ✓      |   ✓   |        |  ✓  |  ✓  |                 |      ✓      |
| Xiaomi MiMo       |            |       |        |  ✓  |     |                 |             |

<Note>
**Realtime spraak** betekent hier bidirectionele realtimefunctionaliteit die eigen is aan de provider (Talk-
modus `realtime`, bijvoorbeeld Gemini Live of de OpenAI Realtime API) — momenteel registreren alleen Google
en OpenAI deze. Deepgram, ElevenLabs, Mistral, OpenAI en xAI
registreren daarnaast afzonderlijk streaming-STT voor Voice Call (eenrichtingsaudio-naar-tekst); zie
[Spraak-naar-tekst en Voice Call](#speech-to-text-and-voice-call) hieronder.
Realtime spraak van xAI is een upstreammogelijkheid, maar wordt niet geregistreerd in
OpenClaw totdat het gedeelde contract voor realtime spraak deze kan vertegenwoordigen.
</Note>

## Asynchroon versus synchroon

| Mogelijkheid     | Modus       | Waarom                                                                                                    |
| ---------------- | ----------- | --------------------------------------------------------------------------------------------------------- |
| Afbeelding       | Asynchroon  | Providerverwerking kan langer duren dan een chatbeurt; gegenereerde bijlagen gebruiken het gedeelde voltooiingspad. |
| Tekst-naar-spraak | Synchroon  | Providerantwoorden worden binnen enkele seconden geretourneerd en aan de antwoordaudio toegevoegd.        |
| Video            | Asynchroon  | Providerverwerking duurt 30 s tot enkele minuten; trage wachtrijen kunnen tot de geconfigureerde time-out actief blijven. |
| Muziek           | Asynchroon  | Heeft dezelfde providerverwerkingseigenschappen als video.                                                |

Voor asynchrone tools dient OpenClaw het verzoek in bij de provider, retourneert het onmiddellijk een taak-
id en volgt het de taak in het taaklogboek. De agent blijft
op andere berichten reageren terwijl de taak wordt uitgevoerd. Wanneer de provider klaar is,
wekt OpenClaw de agent met de paden naar de gegenereerde media, zodat deze de
gebruiker via de normale zichtbare antwoordmodus van de sessie kan informeren: automatische aflevering van het definitieve antwoord
wanneer dit is geconfigureerd, of `message(action="send")` wanneer de sessie
de berichtentool vereist. Als de sessie van de aanvrager inactief is of het actief wekken
mislukt, en er nog gegenereerde media in het voltooiingsantwoord ontbreken,
verstuurt OpenClaw een idempotente directe fallback met uitsluitend de ontbrekende media. Media
die al via het voltooiingsantwoord zijn afgeleverd, worden niet opnieuw geplaatst.

## Spraak-naar-tekst en Voice Call

Deepgram, DeepInfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter,
SenseAudio en xAI kunnen allemaal inkomende audio via het batchpad
`tools.media.audio` transcriberen wanneer ze zijn geconfigureerd. Kanaalplugins die voorafgaand een
spraaknotitie controleren voor vermeldingsfiltering of opdrachtparsing, markeren de getranscribeerde
bijlage in de inkomende context, zodat de gedeelde mediabegripsstap
dat transcript hergebruikt in plaats van voor dezelfde
audio een tweede STT-aanroep uit te voeren.

Deepgram, ElevenLabs, Mistral, OpenAI en xAI registreren ook
streaming-STT-providers voor Voice Call, zodat live telefoonaudio naar de geselecteerde
leverancier kan worden doorgestuurd zonder op een voltooide opname te wachten.

Gebruik voor live gesprekken met gebruikers bij voorkeur de [Talk-modus](/nl/nodes/talk). Batchaudio-
bijlagen blijven op het mediapad; browser-realtime, native push-to-talk,
telefonie en vergaderaudio moeten Talk-events en de sessiegebonden
catalogi gebruiken die door de Gateway worden geretourneerd.

## Providertoewijzingen (hoe leveranciers over oppervlakken zijn verdeeld)

<AccordionGroup>
  <Accordion title="Google">
    Oppervlakken voor afbeeldingen, video, muziek, batch-TTS, batch-STT, backend-realtime spraak en
    mediabegrip.
  </Accordion>
  <Accordion title="OpenAI">
    Oppervlakken voor afbeeldingen, video, batch-TTS, batch-STT, streaming-STT voor Voice Call, backend-
    realtime spraak en geheugen-embeddings.
  </Accordion>
  <Accordion title="DeepInfra">
    Oppervlakken voor chat-/modelroutering, het genereren/bewerken van afbeeldingen, tekst-naar-video, batch-TTS,
    batch-STT, mediabegrip voor afbeeldingen en geheugen-embeddings.
    DeepInfra biedt ook herrangschikking, classificatie, objectdetectie en
    andere native modeltypen; OpenClaw heeft nog geen providercontract voor deze
    categorieën, dus deze plugin registreert ze niet.
  </Accordion>
  <Accordion title="xAI">
    Afbeeldingen, video, zoeken, code-uitvoering, batch-TTS, batch-STT en streaming-STT voor Voice
    Call. Realtime spraak van xAI is een upstreammogelijkheid, maar wordt
    niet geregistreerd in OpenClaw totdat het gedeelde contract voor realtime spraak deze kan
    vertegenwoordigen.
  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Afbeeldingen genereren](/nl/tools/image-generation)
- [Video's genereren](/nl/tools/video-generation)
- [Muziek genereren](/nl/tools/music-generation)
- [Tekst-naar-spraak](/nl/tools/tts)
- [Mediabegrip](/nl/nodes/media-understanding)
- [Audionodes](/nl/nodes/audio)
- [Talk-modus](/nl/nodes/talk)
