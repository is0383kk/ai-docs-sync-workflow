---
read_when:
    - Je wijzigt de Markdown-opmaak of segmentering voor uitgaande kanalen
    - Je voegt een nieuwe kanaalformatter of stijltoewijzing toe
    - Je debugt opmaakregressies in verschillende kanalen
summary: Markdown-opmaakpijplijn voor uitgaande kanalen
title: Markdown-opmaak
x-i18n:
    generated_at: "2026-07-27T04:57:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9a35fd9a6386068e1e3bec73ec6e692f49239b468f42dd737f919b1c6a88e41
    source_path: concepts/markdown-formatting.md
    workflow: 16
---

OpenClaw zet uitgaande Markdown om in een gedeelde tussenrepresentatie
(IR) voordat kanaalspecifieke uitvoer wordt gerenderd. De IR bevat platte tekst plus
stijl-/linkbereiken, zodat één parseerstap elk kanaal bedient en het opdelen in chunks
de opmaak nooit midden in een bereik splitst.

## Pijplijn

1. **Markdown naar IR parsen** (`markdownToIR`) - platte tekst plus stijlbereiken
   (vet, cursief, doorgestreept, code, codeblok, spoiler, blokcitaat,
   kop 1-6) en linkbereiken. Offsets zijn UTF-16-code-eenheden, zodat de stijlbereiken van Signal
   rechtstreeks aansluiten op de API. Tabellen worden alleen geparseerd wanneer het kanaal
   een tabelmodus inschakelt.
2. **De IR opdelen in chunks** (`chunkMarkdownIR` / `renderMarkdownIRChunksWithinLimit`)
   - het splitsen gebeurt vóór het renderen op de IR-tekst, zodat inline stijlen en
     links per chunk worden opgesplitst in plaats van over een grens heen te breken.
3. **Per kanaal renderen** (`renderMarkdownWithMarkers`) - een toewijzing van stijlmarkeringen
   zet bereiken om in de eigen opmaak van het kanaal.

| Kanaal                                                           | Renderer                                                                             | Opmerkingen                                                                               |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Slack                                                            | mrkdwn-tokens (`*bold*`, `_italic_`, `` `code` ``, code fences)                     | Links worden `<url\|label>`; autolink is tijdens het parsen uitgeschakeld om dubbele links te voorkomen |
| Telegram                                                         | HTML-tags (`<b>`, `<i>`, `<s>`, `<code>`, `<pre><code>`, `<a href>`, `<tg-spoiler>`) | Ondersteunt ook tabellen en koppen in uitgebreide berichten (`<h1>`-`<h6>`) wanneer `richMessages` is ingeschakeld |
| Signal                                                           | platte tekst + `text-style`-bereiken                                                   | Links worden gerenderd als `label (url)` wanneer het label afwijkt van de URL             |
| Discord, WhatsApp, iMessage, Microsoft Teams en andere kanalen   | platte tekst                                                                          | Geen op IR gebaseerde opmaak; conversie van Markdown-tabellen wordt nog steeds uitgevoerd via `convertMarkdownTables` |

## IR-voorbeeld

Markdown-invoer:

```markdown
Hallo **wereld** - bekijk de [documentatie](https://docs.openclaw.ai).
```

IR (schematisch):

```json
{
  "text": "Hallo wereld - bekijk de documentatie.",
  "styles": [{ "start": 6, "end": 12, "style": "bold" }],
  "links": [{ "start": 27, "end": 39, "href": "https://docs.openclaw.ai" }]
}
```

## Tabelverwerking

`markdown.tables` bepaalt hoe een kanaal Markdown-tabellen converteert, per
kanaal en optioneel per account:

| Modus     | Gedrag                                                                               |
| --------- | ------------------------------------------------------------------------------------ |
| `code`    | Renderen als een uitgelijnde ASCII-tabel in een codeblok (standaardterugvaloptie)    |
| `bullets` | Elke rij omzetten in `label: value`-opsommingstekens                                 |
| `block`   | Eigen tabellen behouden waar het transport ze ondersteunt; anders terugvallen op `code` |
| `off`     | Het parsen van tabellen uitschakelen; onbewerkte tabeltekst wordt ongewijzigd doorgegeven |

Standaardwaarden van plugins per kanaal: Signal, WhatsApp en Matrix gebruiken standaard
`bullets`; Mattermost gebruikt standaard `off`; Telegram gebruikt standaard `block` (wat
wordt omgezet in `code`, tenzij `richMessages` voor het account is ingeschakeld). Elk
kanaal zonder een expliciete standaardwaarde van de plugin valt terug op `code`.

```yaml
channels:
  discord:
    markdown:
      tables: code
    accounts:
      work:
        markdown:
          tables: off
```

## Regels voor het opdelen in chunks

- Chunklimieten zijn afkomstig van kanaaladapters/configuratie en gelden voor IR-tekst, niet voor
  gerenderde uitvoer.
- Omheinde codeblokken worden als één blok behouden met een afsluitende nieuwe regel, zodat
  kanalen de afsluitende omheining correct renderen.
- Voorvoegsels van lijsten en blokcitaten maken deel uit van de IR-tekst, zodat het opdelen in chunks nooit
  midden in een voorvoegsel splitst.
- Inline stijlen worden nooit over chunks gesplitst; de renderer heropent een geopende
  stijl aan het begin van de volgende chunk.

Zie [Streaming en opdelen in chunks](/concepts/streaming) voor chunkgrenzen en
aflevergedrag voor verschillende kanalen.

## Linkbeleid

- **Slack:** `[label](url)` -> `<url|label>`; kale URL's blijven kaal.
- **Telegram:** `[label](url)` -> `<a href="url">label</a>` (HTML-parseermodus).
- **Signal:** `[label](url)` -> `label (url)`, tenzij het label al
  overeenkomt met de URL.

## Spoilers

Spoilermarkeringen (`||spoiler||`) worden geparseerd voor Signal (toegewezen aan
`SPOILER`-stijlbereiken) en Telegram (toegewezen aan `<tg-spoiler>`). Andere kanalen behandelen
`||...||` als platte tekst.

## Een kanaalformatter toevoegen of bijwerken

1. **Eenmaal parsen** met `markdownToIR(...)`, met voor het kanaal geschikte
   opties (`autolink`, `headingStyle`, `blockquotePrefix`, `tableMode`).
2. **Renderen** met `renderMarkdownWithMarkers(...)` en een toewijzing van stijlmarkeringen (of
   aangepaste stijlbereiklogica voor transportsystemen zoals Signal).
3. **Opdelen in chunks** met `chunkMarkdownIR(...)` of
   `renderMarkdownIRChunksWithinLimit(...)` voordat elke chunk wordt gerenderd.
4. **De adapter koppelen** zodat deze de nieuwe chunker en renderer aanroept vanuit het
   uitgaande verzendpad.
5. **Testen** met opmaaktests plus een test voor uitgaande aflevering als het kanaal
   chunks gebruikt.

## Veelvoorkomende valkuilen

- Slack-tokens met punthaken (`<@U123>`, `<#C123>`, `<https://...>`) moeten
  het escapen overleven; onbewerkte HTML moet nog steeds veilig worden geëscapet.
- Voor Telegram-HTML moet tekst buiten tags worden geëscapet om gebroken opmaak te voorkomen.
- Signal-stijlbereiken gebruiken UTF-16-offsets, geen codepuntafwijkingen.
- Behoud afsluitende nieuwe regels bij omheinde codeblokken, zodat de afsluitende markering
  op een eigen regel terechtkomt.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Streaming en opdelen in chunks" href="/nl/concepts/streaming" icon="bars-staggered">
    Gedrag van uitgaande streaming, chunkgrenzen en kanaalspecifieke aflevering.
  </Card>
  <Card title="Systeemprompt" href="/nl/concepts/system-prompt" icon="message-lines">
    Wat het model vóór het gesprek ziet, inclusief geïnjecteerde werkruimtebestanden.
  </Card>
</CardGroup>
