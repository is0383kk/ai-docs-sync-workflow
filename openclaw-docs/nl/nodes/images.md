---
read_when:
    - Mediapijplijn of bijlagen wijzigen
summary: Regels voor de verwerking van afbeeldingen en media bij verzenden, Gateway en agentantwoorden
title: Ondersteuning voor afbeeldingen en media
x-i18n:
    generated_at: "2026-07-27T05:37:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71f5591f4268593c142056370802b702899787a79f9ca1fbde6ea8e422f34023
    source_path: nodes/images.md
    workflow: 16
---

Het WhatsApp-kanaal draait op Baileys Web. Deze pagina beschrijft de regels voor mediaverwerking bij verzendingen, via de Gateway en in antwoorden van de agent.

## Doelen

- Media verzenden met een optioneel bijschrift via `openclaw message send --media`.
- Automatische antwoorden vanuit de webinbox toestaan om naast tekst ook media te bevatten.
- Limieten per type redelijk en voorspelbaar houden.

## CLI-oppervlak

`openclaw message send --target <dest> --media <path-or-url> [--message <caption>]`

- `--media <path-or-url>` — media bijvoegen (afbeelding/audio/video/document); accepteert lokale paden of URL's. Optioneel; het bijschrift mag leeg zijn voor verzendingen met alleen media.
- `--gif-playback` — videomedia afspelen als GIF (alleen WhatsApp).
- `--force-document` — media als document verzenden om compressie door het kanaal te voorkomen (Telegram, WhatsApp); geldt voor afbeeldingen, GIF's en video's.
- `--reply-to <id>`, `--thread-id <id>`, `--pin`, `--silent` — opties voor aflevering en threads die worden gedeeld met verzendingen met alleen tekst.
- `--dry-run` — de opgeloste payload weergeven en verzending overslaan.
- `--json` — het resultaat als JSON weergeven: `{ action, channel, dryRun, handledBy, messageId?, payload }` (`payload` bevat het kanaalspecifieke verzendresultaat, inclusief eventuele mediaverwijzing).

## Gedrag van het WhatsApp-webkanaal

- Invoer: lokaal bestandspad **of** HTTP(S)-URL.
- Proces: laden in een buffer, het mediatype detecteren en vervolgens per type de uitgaande payload samenstellen:
  - **Afbeeldingen:** geoptimaliseerd om onder `channels.whatsapp.mediaMaxMb` te blijven (standaard 50MB). Ondoorzichtige afbeeldingen worden opnieuw gecomprimeerd als JPEG (de standaardreeks voor zijden begint bij 2048px en loopt af wanneer de maximale grootte herhaaldelijk wordt overschreden); afbeeldingen met transparantie blijven PNG. Als de bron al een geschikte JPEG/PNG/WebP is die binnen het budget voor grootte en zijlengte valt, blijven de oorspronkelijke bytes ongewijzigd behouden in plaats van opnieuw te worden gecomprimeerd. Geanimeerde GIF's worden nooit opnieuw gecodeerd; alleen de grootte wordt gecontroleerd.
  - **Audio/spraak:** tenzij de audio al een systeemeigen spraakindeling heeft (`.ogg`/`.opus` of `audio/ogg`/`audio/opus`), wordt uitgaande audio vóór verzending als spraakbericht (`ptt: true`) via `ffmpeg` getranscodeerd naar Opus/OGG (48kHz mono, 64kbps, maximaal 20 minuten).
  - **Video:** ongewijzigd doorsturen tot 16MB.
  - **Documenten:** al het overige, tot 100MB, waarbij de bestandsnaam indien beschikbaar behouden blijft.
- Afspelen in WhatsApp als GIF: verzend een MP4 met `gifPlayback: true` (CLI: `--gif-playback`), zodat mobiele clients deze herhaaldelijk inline afspelen.
- MIME-detectie geeft de voorkeur aan herkende magische bytes, vervolgens aan de bestandsextensie en daarna aan responsheaders; een algemeen herkende container (`application/octet-stream`, `zip`) overschrijft nooit een specifiekere extensietoewijzing (bijvoorbeeld XLSX tegenover ZIP).
- Het bijschrift is afkomstig van `--message` of `reply.text`; een leeg bijschrift is toegestaan.
- Logboekregistratie: zonder uitgebreide uitvoer worden `↩️`/`✅` weergegeven; uitgebreide uitvoer bevat de grootte en het bronpad/de bron-URL.

<Note>
De bovenstaande waarden van 16MB voor audio/video en 100MB voor documenten zijn de gedeelde standaardlimieten per mediatype wanneer geen expliciete bytelimiet wordt opgegeven. WhatsApp-verzendingen stellen een expliciete limiet in via `channels.whatsapp.mediaMaxMb` (standaard 50MB), die voor dat account uniform op alle typen wordt toegepast.
</Note>

## Pijplijn voor automatische antwoorden

- `getReplyFromConfig` retourneert een antwoordpayload (of een array van payloads) met onder andere `text?`, `mediaUrl?` en `mediaUrls?`.
- Wanneer media aanwezig zijn, lost de webverzender lokale paden of URL's op met dezelfde pijplijn als `openclaw message send`.
- Als meerdere media-items worden opgegeven, worden deze achtereenvolgens verzonden.

## Inkomende media voor opdrachten

- Wanneer inkomende webberichten media bevatten, downloadt OpenClaw deze naar een tijdelijk bestand en stelt het sjabloonvariabelen beschikbaar:
  - `{{AttachmentUrl}}` — oorspronkelijke URL of providerverwijzing voor de huidige bijlage.
  - `{{AttachmentPath}}` — lokaal tijdelijk pad dat wordt geschreven voordat de opdracht wordt uitgevoerd.
  - `{{AttachmentContentType}}` — MIME-inhoudstype.
  - `{{AttachmentDir}}` — map die het lokale pad bevat.
  - `{{AttachmentIndex}}` — op nul gebaseerde index van het bronfeit.
- Wanneer een Docker-sandbox per sessie is ingeschakeld, worden inkomende media naar de werkruimte van de sandbox gekopieerd en wordt het pad/de verwijzing van de bijlage herschreven naar een sandboxrelatief pad, zoals `media/inbound/<filename>`.
- `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` en `{{MediaDir}}` blijven tijdens de migratieperiode van de Plugin-SDK verouderde compatibiliteitsaliassen.
- Mediabegrip (geconfigureerd via `tools.media.*` of gedeeld `tools.media.models`) wordt vóór de sjabloonverwerking uitgevoerd en kan blokken met `[Image]`, `[Audio]` en `[Video]` invoegen in `Body`.
  - Audio stelt `{{Transcript}}` in en gebruikt het transcript voor het verwerken van opdrachten, zodat slash-opdrachten blijven werken.
  - Bij video- en afbeeldingsbeschrijvingen blijft eventuele bijschrifttekst behouden voor het verwerken van opdrachten.
  - Als het actieve primaire model al systeemeigen ondersteuning voor beeld heeft, slaat OpenClaw het samenvattingsblok `[Image]` over en geeft het in plaats daarvan de oorspronkelijke afbeelding door aan het model.
- Standaard wordt alleen de eerste overeenkomende afbeelding/audio-/videobijlage verwerkt; gebruik `tools.media.<capability>.attachments` om meerdere bijlagen te selecteren.

## Limieten en fouten

**Limieten voor uitgaande verzendingen (verzending via WhatsApp-web)**

- Afbeeldingen: na optimalisatie maximaal `channels.whatsapp.mediaMaxMb` (standaard 50MB).
- Audio/video: limiet van 16MB (gedeelde standaardwaarde; bij verzending via WhatsApp overschreven door `mediaMaxMb`).
- Documenten: limiet van 100MB (gedeelde standaardwaarde; bij verzending via WhatsApp overschreven door `mediaMaxMb`).
- Te grote of onleesbare media leveren een duidelijke foutmelding in de logboeken op en het antwoord wordt overgeslagen.

**Limieten voor mediabegrip (transcriptie/beschrijving)**

- Standaard voor afbeeldingen: 10MB (overschrijf met `tools.media.image.maxBytes` of per
  `tools.media.models[]`-item met `maxBytes`).
- Standaard voor audio: 20MB (overschrijf met `tools.media.audio.maxBytes` of per item).
- Standaard voor video: 50MB (overschrijf met `tools.media.video.maxBytes` of per item).
- Bij te grote media wordt het mediabegrip overgeslagen, maar het antwoord wordt nog steeds met de oorspronkelijke hoofdtekst verwerkt.

## Opmerkingen voor tests

- Test de verzend- en antwoordprocessen voor afbeeldingen, audio en documenten.
- Controleer na afbeeldingsoptimalisatie de groottelimieten en voor audio de vlag voor spraakberichten.
- Zorg dat antwoorden met meerdere media-items worden opgesplitst in opeenvolgende verzendingen.

## Gerelateerd

- [Camera-opname](/nl/nodes/camera)
- [Mediabegrip](/nl/nodes/media-understanding)
- [Audio en spraakberichten](/nl/nodes/audio)
