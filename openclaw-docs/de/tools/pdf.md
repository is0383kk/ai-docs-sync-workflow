---
read_when:
    - Sie möchten PDFs von Agenten analysieren
    - Sie benötigen die genauen Parameter und Limits des PDF-Tools
    - Sie debuggen den nativen PDF-Modus im Vergleich zum Extraktions-Fallback.
summary: Analysieren Sie ein oder mehrere PDF-Dokumente mit nativer Provider-Unterstützung und Extraktions-Fallback
title: PDF-Tool
x-i18n:
    generated_at: "2026-07-26T18:50:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0e5b897e1e122af4b2f6f9a3eaeb73f6e93af1051d306ad82539b258de90c49
    source_path: tools/pdf.md
    workflow: 16
---

`pdf` analysiert ein oder mehrere PDF-Dokumente und gibt Text zurück. Das Tool verwendet die native Dokumenteingabe bei Anthropic- und Google-Modellen und greift bei allen anderen Providern auf die Text-/Bildextraktion zurück.

## Verfügbarkeit

Das Tool wird nur registriert, wenn OpenClaw ein PDF-fähiges Modell für den Agenten auflösen kann. Auflösungsreihenfolge:

1. `agents.defaults.pdfModel` (explizites primäres Modell/Fallbacks)
2. `agents.defaults.imageModel` (explizites primäres Modell/Fallbacks)
3. Das aufgelöste Sitzungs-/Standardmodell des Agenten, wenn dessen Provider native PDF-Eingaben unterstützt (Anthropic, Google) oder bereits ein Vision-Modell konfiguriert ist
4. Automatisch erkannte bild-/visionfähige Provider mit verwendbarer Authentifizierung, wobei Provider mit nativer PDF-Unterstützung bevorzugt werden

Die Authentifizierung jedes Fallback-Kandidaten wird vor der Verwendung geprüft. Daher zählt ein konfiguriertes `provider/model` nur, wenn OpenClaw diesen Provider für den Agenten authentifizieren kann. Wenn kein verwendbares Modell aufgelöst werden kann, wird das Tool `pdf` nicht bereitgestellt.

## Eingabereferenz

<ParamField path="pdf" type="string">
Ein PDF-Pfad oder eine PDF-URL.
</ParamField>

<ParamField path="pdfs" type="string[]">
Mehrere PDF-Pfade oder -URLs, insgesamt bis zu 10.
</ParamField>

<ParamField path="prompt" type="string" default="Analyze this PDF document.">
Analyse-Prompt.
</ParamField>

<ParamField path="pages" type="string">
Seitenfilter wie `1-5` oder `1,3,7-9`. Wird im nativen Provider-Modus nicht unterstützt.
</ParamField>

<ParamField path="password" type="string">
Passwort für verschlüsselte PDFs. Gilt für jedes PDF in der Anfrage und wird nur im Extraktions-Fallback-Modus verwendet.
</ParamField>

<ParamField path="model" type="string">
Optionale Modellüberschreibung im Format `provider/model`.
</ParamField>

<ParamField path="maxBytesMb" type="number">
Größenbeschränkung pro PDF in MB. Standardmäßig `agents.defaults.pdfMaxMb` oder `10`, falls nicht festgelegt.
</ParamField>

Hinweise:

- `pdf` und `pdfs` werden vor dem Laden zusammengeführt und dedupliziert; mindestens eines davon ist erforderlich.
- `pages` wird als 1-basierte Seitennummern ausgewertet, dedupliziert, sortiert und auf `agents.defaults.pdfMaxPages` begrenzt (Standardwert: `20`). Ein Bereich, der keine innerhalb der Grenzen liegenden Seiten umfasst, führt vor dem Modellaufruf zu einem Fehler.

## Unterstützte PDF-Referenzen

- Lokaler Dateipfad (einschließlich Erweiterung von `~`)
- `file://`-URL
- `http://`- und `https://`-URL
- Von OpenClaw verwaltete eingehende Referenzen wie `media://inbound/<id>`

Andere URI-Schemata (zum Beispiel `ftp://`) geben `details.error = "unsupported_pdf_reference"` zurück. Entfernte `http(s)`-URLs werden abgelehnt, wenn das Tool in einer Sandbox ausgeführt wird. Bei aktivierter Dateirichtlinie, die den Zugriff auf den Arbeitsbereich beschränkt, werden lokale Pfade außerhalb der zulässigen Stammverzeichnisse abgelehnt; verwaltete eingehende Referenzen und wiedergegebene Pfade im Speicher für eingehende Medien von OpenClaw bleiben zulässig.

## Ausführungsmodi

### Nativer Provider-Modus

Wird für die Provider `anthropic` und `google` verwendet (die einzigen Provider, die derzeit native Unterstützung für PDF-Dokumente deklarieren). Die unverarbeiteten PDF-Bytes werden pro Datei direkt als natives Dokument bzw. Inline-PDF-Teil an die Provider-API gesendet.

Einschränkungen:

- `pages` wird nicht unterstützt; wenn festgelegt, löst das Tool `pages is not supported with native PDF providers` aus.
- `password` wird nicht unterstützt; wenn festgelegt, löst das Tool `password is not supported with native PDF providers` aus. Verwenden Sie für verschlüsselte PDFs ein nicht natives Modell.

### Extraktions-Fallback-Modus

Wird für jeden anderen Provider verwendet.

1. Extrahiert Text aus den ausgewählten Seiten (bis zu `agents.defaults.pdfMaxPages`, Standardwert: `20`) über das mitgelieferte Plugin `document-extract`, das das Paket `clawpdf` (PDFium WebAssembly) zur Text- und Bildextraktion verwendet.
2. Wenn der extrahierte Text kürzer als `200` Zeichen ist, werden dieselben Seiten als PNG-Bilder gerendert. Das Renderbudget beträgt insgesamt `4,000,000` Pixel und wird auf alle Seiten verteilt, für die Bilder benötigt werden (proportional pro verbleibender Seite, nicht pro Seite). Daher wird das Rendering für Textseiten, die bereits genügend Text enthalten, vollständig übersprungen.
3. Sendet den extrahierten Text (und alle gerenderten Bilder) zusammen mit dem Prompt an das ausgewählte Modell.

Details:

- Verschlüsselte PDFs werden mit dem `password`-Parameter der obersten Ebene geöffnet.
- Wenn das Modell keine Bildeingabe unterstützt und kein Text extrahiert werden kann, gibt das Tool einen Fehler aus.
- Wenn das Rendern der Bilder fehlschlägt, verwirft OpenClaw die Bilder und fährt mit dem extrahierten Text fort.
- Wenn das Zielmodell ausschließlich Text unterstützt und bei der Extraktion Bilder erzeugt wurden, verwirft OpenClaw die Bilder und sendet nur den Text.

## Konfiguration

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

| Schlüssel                      | Standardwert       | Bedeutung                                                                                             |
| ----------------------------- | ------- | ----------------------------------------------------------------------------------------- |
| `agents.defaults.pdfModel`    | nicht festgelegt   | Explizite primäre PDF-Modelle/Fallbacks; greift auf `imageModel` und anschließend auf das Sitzungsmodell zurück. |
| `agents.defaults.pdfMaxMb`    | `10`    | Größenbeschränkung pro PDF in MB.                                                                   |
| `agents.defaults.pdfMaxPages` | `20`    | Maximale Anzahl verarbeiteter Seiten pro PDF.                                                              |

Vollständige Felddetails finden Sie in der [Konfigurationsreferenz](/de/gateway/config-agents#agent-defaults).

## Ausgabedetails

Das Tool gibt Text in `content[0].text` und strukturierte Metadaten in `details` zurück.

Häufig verwendete `details`-Felder:

- `model`: aufgelöste Modellreferenz (`provider/model`)
- `native`: `true` für den nativen Provider-Modus, `false` für den Fallback
- `attempts`: fehlgeschlagene Fallback-Versuche vor dem Erfolg

Pfadfelder:

- Einzelne PDF-Eingabe: `details.pdf`
- Mehrere PDF-Eingaben: `details.pdfs[]` mit `pdf`-Einträgen
- Metadaten zur Pfadumschreibung in der Sandbox (falls zutreffend): `rewrittenFrom`

## Fehlerverhalten

| Bedingung                         | Ergebnis                                                         |
| --------------------------------- | -------------------------------------------------------------- |
| Keine PDF-Eingabe                 | Löst `pdf required: provide a path or URL to a PDF document` aus |
| Mehr als 10 PDFs                  | `details.error = "too_many_pdfs"`                              |
| Nicht unterstütztes Referenzschema | `details.error = "unsupported_pdf_reference"`                  |
| `pages` mit einem nativen Provider    | Löst `pages is not supported with native PDF providers` aus      |
| `password` mit einem nativen Provider | Löst `password is not supported with native PDF providers` aus   |

## Beispiele

Einzelnes PDF:

```json
{
  "pdf": "/tmp/report.pdf",
  "prompt": "Fassen Sie diesen Bericht in 5 Stichpunkten zusammen"
}
```

Mehrere PDFs:

```json
{
  "pdfs": ["/tmp/q1.pdf", "/tmp/q2.pdf"],
  "prompt": "Vergleichen Sie Risiken und Änderungen am Zeitplan in beiden Dokumenten"
}
```

Fallback-Modell mit Seitenfilter:

```json
{
  "pdf": "https://example.com/report.pdf",
  "pages": "1-3,7",
  "model": "openai/gpt-5.4-mini",
  "prompt": "Extrahieren Sie nur Vorfälle mit Auswirkungen auf Kunden"
}
```

Verschlüsseltes PDF mit Extraktions-Fallback:

```json
{
  "pdf": "/tmp/locked.pdf",
  "password": "example-password",
  "model": "openai/gpt-5.4-mini",
  "prompt": "Fassen Sie diesen Vertrag zusammen"
}
```

## Verwandte Themen

- [Tool-Übersicht](/de/tools) – alle verfügbaren Agenten-Tools
- [Konfigurationsreferenz](/de/gateway/config-agents#agent-defaults) – Konfiguration von pdfMaxBytesMb und pdfMaxPages
