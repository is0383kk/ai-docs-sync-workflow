---
read_when:
    - Je wilt PDF's van agents analyseren
    - Je hebt exacte parameters en limieten voor de pdf-tool nodig
    - Je debugt de native PDF-modus versus de fallback voor extractie
summary: Analyseer een of meer PDF-documenten met native providerondersteuning en extractie als terugvaloptie
title: PDF-tool
x-i18n:
    generated_at: "2026-07-27T06:16:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0e5b897e1e122af4b2f6f9a3eaeb73f6e93af1051d306ad82539b258de90c49
    source_path: tools/pdf.md
    workflow: 16
---

`pdf` analyseert een of meer PDF-documenten en retourneert tekst. De tool gebruikt native documentinvoer bij Anthropic- en Google-modellen en valt bij alle andere providers terug op tekst-/afbeeldingsextractie.

## Beschikbaarheid

De tool wordt alleen geregistreerd wanneer OpenClaw een PDF-geschikt model voor de agent kan vinden. Zoekvolgorde:

1. `agents.defaults.pdfModel` (expliciete primaire modellen/terugvalmodellen)
2. `agents.defaults.imageModel` (expliciete primaire modellen/terugvalmodellen)
3. Het gevonden sessie-/standaardmodel van de agent, als de provider native PDF-invoer ondersteunt (Anthropic, Google) of al een geconfigureerd visionmodel heeft
4. Automatisch gedetecteerde providers die afbeeldingen/vision ondersteunen en bruikbare authenticatie hebben, waarbij providers met native PDF-ondersteuning voorrang krijgen

Voor gebruik wordt de authenticatie van elke kandidaat voor terugval gecontroleerd. Een geconfigureerde `provider/model` telt dus alleen mee als OpenClaw de agent bij die provider kan authenticeren. Als er geen bruikbaar model wordt gevonden, wordt de tool `pdf` niet beschikbaar gesteld.

## Invoerverwijzing

<ParamField path="pdf" type="string">
Eén PDF-pad of één PDF-URL.
</ParamField>

<ParamField path="pdfs" type="string[]">
Meerdere PDF-paden of -URL's, maximaal 10 in totaal.
</ParamField>

<ParamField path="prompt" type="string" default="Analyze this PDF document.">
Analyseprompt.
</ParamField>

<ParamField path="pages" type="string">
Paginafilter zoals `1-5` of `1,3,7-9`. Niet ondersteund in de native providermodus.
</ParamField>

<ParamField path="password" type="string">
Wachtwoord voor versleutelde PDF's. Geldt voor elke PDF in het verzoek; wordt alleen gebruikt in de extractieterugvalmodus.
</ParamField>

<ParamField path="model" type="string">
Optionele modeloverschrijving in de vorm `provider/model`.
</ParamField>

<ParamField path="maxBytesMb" type="number">
Groottelimiet per PDF in MB. Standaard `agents.defaults.pdfMaxMb`, of `10` indien niet ingesteld.
</ParamField>

Opmerkingen:

- `pdf` en `pdfs` worden vóór het laden samengevoegd en gededupliceerd; ten minste één ervan is vereist.
- `pages` wordt geïnterpreteerd als paginanummers die bij 1 beginnen, gededupliceerd, gesorteerd en begrensd tot `agents.defaults.pdfMaxPages` (standaard `20`). Een bereik dat geen enkele pagina binnen de grenzen bevat, veroorzaakt vóór de modelaanroep een fout.

## Ondersteunde PDF-verwijzingen

- Lokaal bestandspad (inclusief uitbreiding van `~`)
- `file://`-URL
- `http://`- en `https://`-URL
- Door OpenClaw beheerde inkomende verwijzingen, zoals `media://inbound/<id>`

Andere URI-schema's (bijvoorbeeld `ftp://`) retourneren `details.error = "unsupported_pdf_reference"`. Externe `http(s)`-URL's worden geweigerd wanneer de tool in een sandbox wordt uitgevoerd. Als het bestandsbeleid voor alleen de workspace is ingeschakeld, worden lokale paden buiten de toegestane hoofdmappen geweigerd; beheerde inkomende verwijzingen en opnieuw afgespeelde paden in OpenClaws opslag voor inkomende media blijven toegestaan.

## Uitvoeringsmodi

### Native providermodus

Gebruikt voor provider `anthropic` en `google` (de enige providers die momenteel native ondersteuning voor PDF-documenten declareren). De onbewerkte PDF-bytes gaan per bestand rechtstreeks naar de provider-API als native document-/inline-PDF-onderdeel.

Limieten:

- `pages` wordt niet ondersteund; indien ingesteld, genereert de tool `pages is not supported with native PDF providers`.
- `password` wordt niet ondersteund; indien ingesteld, genereert de tool `password is not supported with native PDF providers`. Gebruik voor versleutelde PDF's een niet-native model.

### Extractieterugvalmodus

Gebruikt voor alle andere providers.

1. Extraheer tekst uit de geselecteerde pagina's (maximaal `agents.defaults.pdfMaxPages`, standaard `20`) via de meegeleverde Plugin `document-extract`, die het pakket `clawpdf` (PDFium WebAssembly) gebruikt voor tekst- en afbeeldingsextractie.
2. Als de geëxtraheerde tekst korter is dan `200` tekens, worden dezelfde pagina's als PNG-afbeeldingen gerenderd. Het renderbudget bedraagt in totaal `4,000,000` pixels en wordt gedeeld door alle pagina's waarvoor afbeeldingen nodig zijn (evenredig toegewezen per resterende pagina, niet per pagina). Tekstpagina's die al voldoende tekst bevatten, worden dus helemaal niet gerenderd.
3. Stuur de geëxtraheerde tekst (en eventuele gerenderde afbeeldingen) samen met de prompt naar het geselecteerde model.

Details:

- Versleutelde PDF's worden geopend met de parameter `password` op het hoogste niveau.
- Als het model geen afbeeldingsinvoer ondersteunt en er geen tekst kan worden geëxtraheerd, genereert de tool een fout.
- Als het renderen van afbeeldingen mislukt, laat OpenClaw de afbeeldingen weg en gaat het verder met de geëxtraheerde tekst.
- Als het doelmodel alleen tekst ondersteunt en de extractie afbeeldingen heeft opgeleverd, laat OpenClaw de afbeeldingen weg en verzendt het alleen tekst.

## Configuratie

```json5
{
  agents: {
    defaults: {
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
    },
  },
}
```

| Sleutel                       | Standaard | Betekenis                                                                                |
| ----------------------------- | --------- | ---------------------------------------------------------------------------------------- |
| `agents.defaults.pdfModel`    | niet ingesteld | Expliciete primaire PDF-modellen/terugvalmodellen; valt terug op `imageModel` en daarna op het sessiemodel. |
| `agents.defaults.pdfMaxMb`    | `10`    | Groottelimiet per PDF in MB.                                                             |
| `agents.defaults.pdfMaxPages` | `20`    | Maximaal aantal verwerkte pagina's per PDF.                                              |

Zie [Configuratiereferentie](/nl/gateway/config-agents#agent-defaults) voor alle veldgegevens.

## Uitvoerdetails

De tool retourneert tekst in `content[0].text` en gestructureerde metagegevens in `details`.

Veelvoorkomende `details`-velden:

- `model`: gevonden modelverwijzing (`provider/model`)
- `native`: `true` voor de native providermodus, `false` voor terugval
- `attempts`: mislukte terugvalpogingen vóór het slagen

Padvelden:

- Invoer van één PDF: `details.pdf`
- Invoer van meerdere PDF's: `details.pdfs[]` met `pdf`-items
- Metagegevens over het herschrijven van sandboxpaden (indien van toepassing): `rewrittenFrom`

## Foutgedrag

| Voorwaarde                        | Resultaat                                                      |
| --------------------------------- | -------------------------------------------------------------- |
| Geen PDF-invoer                   | Genereert `pdf required: provide a path or URL to a PDF document` |
| Meer dan 10 PDF's                 | `details.error = "too_many_pdfs"`                              |
| Niet-ondersteund verwijzingsschema | `details.error = "unsupported_pdf_reference"`                  |
| `pages` met een native provider    | Genereert `pages is not supported with native PDF providers`      |
| `password` met een native provider | Genereert `password is not supported with native PDF providers`   |

## Voorbeelden

Eén PDF:

```json
{
  "pdf": "/tmp/report.pdf",
  "prompt": "Vat dit rapport samen in 5 opsommingstekens"
}
```

Meerdere PDF's:

```json
{
  "pdfs": ["/tmp/q1.pdf", "/tmp/q2.pdf"],
  "prompt": "Vergelijk risico's en wijzigingen in de tijdlijn tussen beide documenten"
}
```

Terugvalmodel met paginafilter:

```json
{
  "pdf": "https://example.com/report.pdf",
  "pages": "1-3,7",
  "model": "openai/gpt-5.4-mini",
  "prompt": "Extraheer alleen incidenten die gevolgen hebben voor klanten"
}
```

Versleutelde PDF met extractieterugval:

```json
{
  "pdf": "/tmp/locked.pdf",
  "password": "example-password",
  "model": "openai/gpt-5.4-mini",
  "prompt": "Vat dit contract samen"
}
```

## Gerelateerd

- [Toolsoverzicht](/nl/tools) - alle beschikbare agenttools
- [Configuratiereferentie](/nl/gateway/config-agents#agent-defaults) - configuratie van pdfMaxBytesMb en pdfMaxPages
