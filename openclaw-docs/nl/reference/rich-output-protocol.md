---
read_when:
    - De weergave van assistentuitvoer in de Control UI wijzigen
    - Foutopsporing voor `[embed ...]`, gestructureerde richtlijnen voor de presentatie van media, antwoorden of audio
summary: Protocol voor rijke uitvoer met gestructureerde media, insluitingen, audioaanwijzingen en antwoorden
title: Protocol voor rijke uitvoer
x-i18n:
    generated_at: "2026-07-27T05:33:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbfe68f38c871f5f6d2811eb52b18d0143606f30283023ae96db64543eed95a1
    source_path: reference/rich-output-protocol.md
    workflow: 16
---

Assistentuitvoer draagt leverings- en renderinstructies over via enkele specifieke kanalen:

- Gestructureerde `mediaUrl`- / `mediaUrls`-velden voor het leveren van bijlagen.
- `[[audio_as_voice]]` voor aanwijzingen voor audioweergave.
- `[[reply_to_current]]` / `[[reply_to:<id>]]` voor antwoordmetadata.
- `[embed ...]` voor rijke rendering in de Control-UI.

Gestructureerde mediavelden en `[[...]]`-tags zijn leveringsmetadata. `[embed ...]` is het afzonderlijke, uitsluitend voor het web bestemde pad voor rijke rendering; het is geen media-alias.

## Mediabijlagen

Externe bijlagen moeten openbare `https:`-URL's zijn. `http:`-, loopback-, link-local-, privé- en interne hostnamen worden geweigerd als bijlage-instructies; mediaservers die inhoud ophalen, passen daarbovenop hun eigen netwerkbeveiligingen toe.

Lokale bijlagen accepteren absolute paden, werkruimterelatieve paden of thuisdirectoryrelatieve `~/`-paden. Vóór levering worden ze nog steeds getoetst aan het bestandsleesbeleid van de agent en aan controles van het mediatype.

<Warning>
Genereer vanuit tools, plugins, streamingblokken, browseruitvoer of berichtacties geen tekstopdrachten voor bijlagen. Gebruik in plaats daarvan gestructureerde mediavelden:

```json
{ "message": "Hier is je afbeelding.", "mediaUrl": "/workspace/image.png" }
```

Tekst uit oudere definitieve antwoorden kan voor compatibiliteit nog steeds worden genormaliseerd, maar dit is geen algemeen protocol voor plugins/tools.
</Warning>

Normale Markdown-afbeeldingssyntaxis (`![alt](url)`) blijft standaard tekst. Kanalen die Markdown-afbeeldingen als media-antwoorden willen behandelen, schakelen dit in via hun uitgaande adapter; Telegram doet dit, zodat `![alt](url)` een mediabijlage wordt.

Wanneer blokstreaming is ingeschakeld, moeten media via gestructureerde payloadvelden worden meegestuurd. Als dezelfde media-URL in een gestreamd blok en opnieuw in de definitieve payload van de assistent voorkomt, levert OpenClaw deze eenmaal en verwijdert het duplicaat uit de definitieve payload.

## `[embed ...]`

`[embed ...]` is de enige agentgerichte syntaxis voor rijke rendering in de Control-UI. Zelfsluitend voorbeeld:

```text
[embed ref="cv_123" title="Status" /]
```

Regels:

- `[view ...]` is niet langer geldig voor nieuwe uitvoer.
- Shortcodes voor insluitingen worden alleen in het assistentberichtoppervlak gerenderd.
- Alleen insluitingen met een URL als bron worden gerenderd; gebruik `ref="..."` of `url="..."`.
- Inline HTML-shortcodes voor insluitingen in blokvorm worden niet gerenderd.
- De web-UI verwijdert de shortcode uit de zichtbare tekst en rendert de insluiting inline.

## Opgeslagen renderstructuur

Het genormaliseerde/opgeslagen inhoudsblok van de assistent is een gestructureerd `canvas`-item:

```json
{
  "type": "canvas",
  "preview": {
    "kind": "canvas",
    "surface": "assistant_message",
    "render": "url",
    "viewId": "cv_123",
    "url": "/__openclaw__/canvas/documents/cv_123/index.html",
    "title": "Status",
    "preferredHeight": 320
  }
}
```

`present_view` wordt niet herkend; opgeslagen/gerenderde rijke blokken gebruiken altijd deze `canvas`-structuur.

## Gerelateerd

- [RPC-adapters](/nl/reference/rpc)
- [Typebox](/nl/concepts/typebox)
