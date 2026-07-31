---
read_when:
    - Kanalstandort-Parsing hinzufügen oder ändern
    - Standortkontextfelder in Agenten-Prompts oder Tools verwenden
summary: Parsing von Kanalstandorten und portable ausgehende Standort-Payloads
title: Parsing des Channel-Standorts
x-i18n:
    generated_at: "2026-07-26T17:39:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c7e5647d02643ad6d95024b362228377690d7fdff66441fae367f0f5307217fb
    source_path: channels/location.md
    workflow: 16
---

OpenClaw normalisiert freigegebene Standorte aus Chat-Kanälen in:

- knappen Koordinatentext, der an den eingehenden Nachrichtentext angehängt wird, und
- strukturierte Felder in der Kontext-Nutzlast für automatische Antworten. Vom Kanal bereitgestellte Bezeichnungen, Adressen und Bildunterschriften/Kommentare werden durch den gemeinsamen JSON-Block für nicht vertrauenswürdige Metadaten in den Prompt gerendert, nicht inline in den Nachrichtentext des Benutzers.

Derzeit unterstützt:

- **LINE** (Standortnachrichten mit Titel/Adresse)
- **Matrix** (`m.location` mit `geo_uri`)
- **Telegram** (Standortmarkierungen + Orte + Live-Standorte)
- **WhatsApp** (`locationMessage` + `liveLocationMessage`)

## Textformatierung

Standorte werden als leserfreundliche Zeilen ohne Klammern dargestellt. Koordinaten verwenden sechs Dezimalstellen; die Genauigkeit wird auf ganze Meter gerundet:

- Markierung:
  - `📍 48.858844, 2.294351 ±12m`
- Benannter Ort (in derselben Zeile; Name/Adresse werden nur in den Metadatenblock übernommen):
  - `📍 48.858844, 2.294351 ±12m`
- Live-Freigabe:
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

Wenn der Kanal eine Bezeichnung, Adresse oder Bildunterschrift/einen Kommentar enthält, bleibt diese Angabe in der Kontext-Nutzlast erhalten und erscheint im Prompt als abgegrenztes, nicht vertrauenswürdiges JSON (nicht vorhandene Felder werden ausgelassen):

````text
Standort (nicht vertrauenswürdige Metadaten):
```json
{
  "latitude": 48.858844,
  "longitude": 2.294351,
  "accuracy_m": 12,
  "source": "place",
  "name": "Eiffelturm",
  "address": "Champ de Mars, Paris",
  "caption": "Hier treffen"
}
```
````

## Kontextfelder

Wenn ein Standort vorhanden ist, werden diese Felder zu `ctx` hinzugefügt:

- `LocationLat` (Zahl)
- `LocationLon` (Zahl)
- `LocationAccuracy` (Zahl, Meter; optional)
- `LocationName` (Zeichenfolge; optional)
- `LocationAddress` (Zeichenfolge; optional)
- `LocationSource` (`pin | place | live`)
- `LocationIsLive` (boolescher Wert)
- `LocationCaption` (Zeichenfolge; optional)

Wenn der Kanal keine explizite Quelle festlegt, leitet OpenClaw sie ab: Live-Freigaben werden zu `live`, Standorte mit einem Namen oder einer Adresse zu `place`, alles andere ist `pin`.

Der Prompt-Renderer behandelt `LocationName`, `LocationAddress` und `LocationCaption` als nicht vertrauenswürdige Metadaten und serialisiert sie über denselben begrenzten JSON-Pfad, der für anderen Kanalkontext verwendet wird.

## Ausgehende Nutzlasten

Das Nachrichtenwerkzeug und das Plugin SDK verwenden dieselbe `NormalizedLocation`-Struktur für portable ausgehende Standorte. Eine Nutzlast, die nur Koordinaten enthält, stellt eine Markierung dar. Kanäle mit nativer Unterstützung für Orte können `name` zusammen mit `address` einer Ortskarte zuordnen.

Telegram stellt dies derzeit über `message(action="send")` bereit. Die erste Implementierung ist bewusst eigenständig: Standort-Nutzlasten können nicht mit Text oder Medien kombiniert werden, und unvollständige Ortspaare schlagen fehl, statt einen Namen oder eine Adresse stillschweigend zu verwerfen. Nicht unterstützte Kanäle bieten den Standortparameter nicht an.

## Hinweise zu Kanälen

- **LINE**: Bei Standortnachrichten werden `title`/`address` auf `LocationName`/`LocationAddress` abgebildet; keine Live-Standorte.
- **Matrix**: `geo_uri` wird als Standortmarkierung geparst; der Parameter `u` (Unsicherheit) wird auf `LocationAccuracy` abgebildet, der Ereignistext füllt `LocationCaption` aus, die Höhe wird ignoriert und `LocationIsLive` ist immer falsch.
- **Telegram**: Orte werden auf `LocationName`/`LocationAddress` abgebildet; Live-Standorte werden über `live_period` erkannt.
- **WhatsApp**: `locationMessage.comment` und `liveLocationMessage.caption` füllen `LocationCaption` aus.

## Verwandte Themen

- [Standortbefehl (Nodes)](/de/nodes/location-command)
- [Kameraaufnahme](/de/nodes/camera)
- [Medienverständnis](/de/nodes/media-understanding)
