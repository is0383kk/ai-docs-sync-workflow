---
read_when:
    - Kanaallocatieparsing toevoegen of wijzigen
    - Locatiecontextvelden gebruiken in agentprompts of tools
summary: Parsering van kanaallocaties en overdraagbare uitgaande locatiepayloads
title: Parseren van kanaallocaties
x-i18n:
    generated_at: "2026-07-27T04:47:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c7e5647d02643ad6d95024b362228377690d7fdff66441fae367f0f5307217fb
    source_path: channels/location.md
    workflow: 16
---

OpenClaw normaliseert gedeelde locaties uit chatkanalen naar:

- beknopte coördinatentekst die aan de inkomende berichttekst wordt toegevoegd, en
- gestructureerde velden in de contextpayload voor automatisch antwoorden. Door kanalen aangeleverde labels, adressen en bijschriften/opmerkingen worden in de prompt weergegeven via het gedeelde JSON-blok met niet-vertrouwde metadata, niet rechtstreeks in de berichttekst van de gebruiker.

Momenteel ondersteund:

- **LINE** (locatieberichten met titel/adres)
- **Matrix** (`m.location` met `geo_uri`)
- **Telegram** (locatiespelden + locaties + live locaties)
- **WhatsApp** (`locationMessage` + `liveLocationMessage`)

## Tekstopmaak

Locaties worden weergegeven als leesbare regels zonder haakjes. Coördinaten gebruiken zes decimalen; de nauwkeurigheid wordt afgerond op hele meters:

- Speld:
  - `📍 48.858844, 2.294351 ±12m`
- Benoemde plaats (dezelfde regel; de naam/het adres komen alleen in het metadatablok):
  - `📍 48.858844, 2.294351 ±12m`
- Live delen:
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

Als het kanaal een label, adres of bijschrift/opmerking bevat, blijft dit behouden in de contextpayload en verschijnt het in de prompt als afgeschermde, niet-vertrouwde JSON (velden worden weggelaten als ze ontbreken):

````text
Locatie (niet-vertrouwde metadata):
```json
{
  "latitude": 48.858844,
  "longitude": 2.294351,
  "accuracy_m": 12,
  "source": "place",
  "name": "Eiffeltoren",
  "address": "Champ de Mars, Parijs",
  "caption": "Spreek hier af"
}
```
````

## Contextvelden

Als er een locatie aanwezig is, worden deze velden toegevoegd aan `ctx`:

- `LocationLat` (getal)
- `LocationLon` (getal)
- `LocationAccuracy` (getal, meters; optioneel)
- `LocationName` (tekenreeks; optioneel)
- `LocationAddress` (tekenreeks; optioneel)
- `LocationSource` (`pin | place | live`)
- `LocationIsLive` (booleaanse waarde)
- `LocationCaption` (tekenreeks; optioneel)

Wanneer het kanaal geen expliciete bron instelt, leidt OpenClaw deze af: live gedeelde locaties worden `live`, locaties met een naam of adres worden `place`, en al het overige wordt `pin`.

De promptrenderer behandelt `LocationName`, `LocationAddress` en `LocationCaption` als niet-vertrouwde metadata en serialiseert ze via hetzelfde begrensde JSON-pad dat voor andere kanaalcontext wordt gebruikt.

## Uitgaande payloads

De berichtentool en Plugin SDK gebruiken dezelfde `NormalizedLocation`-structuur voor overdraagbare uitgaande locaties. Een payload met alleen coördinaten vertegenwoordigt een speld. Kanalen met ingebouwde ondersteuning voor locaties kunnen `name` plus `address` omzetten naar een locatiekaart.

Telegram stelt dit momenteel beschikbaar via `message(action="send")`. De eerste implementatie is bewust zelfstandig: locatiepayloads kunnen niet worden gecombineerd met tekst of media, en onvolledige locatieparen mislukken in plaats van stilzwijgend een naam of adres weg te laten. Niet-ondersteunde kanalen bieden de locatieparameter niet aan.

## Opmerkingen per kanaal

- **LINE**: `title`/`address` van locatieberichten worden toegewezen aan `LocationName`/`LocationAddress`; geen live locaties.
- **Matrix**: `geo_uri` wordt als een locatiespeld geparseerd; de parameter `u` (onzekerheid) wordt toegewezen aan `LocationAccuracy`, de berichttekst van de gebeurtenis vult `LocationCaption`, hoogte wordt genegeerd en `LocationIsLive` is altijd false.
- **Telegram**: locaties worden toegewezen aan `LocationName`/`LocationAddress`; live locaties worden gedetecteerd via `live_period`.
- **WhatsApp**: `locationMessage.comment` en `liveLocationMessage.caption` vullen `LocationCaption`.

## Gerelateerd

- [Locatieopdracht (nodes)](/nl/nodes/location-command)
- [Camera-opname](/nl/nodes/camera)
- [Mediabegrip](/nl/nodes/media-understanding)
