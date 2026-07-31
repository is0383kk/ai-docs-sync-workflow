---
read_when:
    - Matrix-clients bouwen die uitgebreide OpenClaw-antwoorden weergeven
    - Foutopsporing in de inhoud van com.openclaw.presentation-events
summary: Matrix MessagePresentation-metadata voor OpenClaw-compatibele clients
title: Matrix-presentatiemetadata
x-i18n:
    generated_at: "2026-07-27T05:02:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0de4d13c6cefc6f91dcc7a4b0edeea6bf001f3bd71f52c9f0498ad422783d8a
    source_path: channels/matrix-presentation.md
    workflow: 16
---

OpenClaw voegt genormaliseerde `MessagePresentation`-metadata toe aan uitgaande Matrix-`m.room.message`-gebeurtenissen onder de inhoudssleutel `com.openclaw.presentation`.

Standaard Matrix-clients blijven de platte tekst `body` weergeven. Clients die OpenClaw ondersteunen, kunnen de gestructureerde metadata lezen en systeemeigen UI weergeven, zoals knoppen, selectielijsten, contextregels en scheidingslijnen.

## Gebeurtenisinhoud

```json
{
  "msgtype": "m.text",
  "body": "Selecteer model\n\nKies model:\n- DeepSeek",
  "com.openclaw.presentation": {
    "version": 1,
    "type": "message.presentation",
    "title": "Selecteer model",
    "tone": "info",
    "blocks": [
      {
        "type": "select",
        "placeholder": "Kies model",
        "options": [
          {
            "label": "DeepSeek",
            "value": "/model deepseek/deepseek-chat"
          }
        ]
      }
    ]
  }
}
```

- `version` is de versie van het metadataschema; de huidige versie is `1`. `type` is een stabiele discriminator, altijd `"message.presentation"`. De Matrix-adapter verzendt alleen payloads met precies deze versie en dit type; clients moeten eveneens onbekende versies negeren die ze niet veilig kunnen interpreteren, evenals onbekende waarden voor `type` en onbekende bloktypen.
- `title` en `tone` (`info`, `success`, `warning`, `danger`, `neutral`) zijn optionele aanwijzingen.
- Knoppen en selectieopties kunnen naast de verouderde tekenreeks `value` een getypeerde `action` (`{ "type": "command", "command": "/..." }` of `{ "type": "callback", "value": "..." }`) bevatten. Geef de voorkeur aan `action` wanneer beide aanwezig zijn.

## Terugvalgedrag

OpenClaw geeft altijd een leesbare terugvaltekst in platte tekst weer in `body`. De gestructureerde metadata is aanvullend en mag niet vereist zijn voor elementaire interoperabiliteit met Matrix.

Regels voor terugvalweergave:

- Inhoud van `title`, `text` en `context` wordt als platte regels weergegeven.
- Knoppen met een `command`-actie worden weergegeven als ``label: `/command` ``, zodat de opdracht kopieerbaar blijft. Knoppen met een `callback`-actie of alleen een verouderde `value` worden uitsluitend met hun label weergegeven, zodat ondoorzichtige callbackwaarden privé blijven; uitgeschakelde knoppen worden altijd uitsluitend met hun label weergegeven. URL- en webappknoppen worden weergegeven als `label: URL`.
- Selectieblokken geven de tijdelijke aanduiding (of `Options:`) weer als kop, gevolgd door optieregels met alleen labels.
- Als niets wordt weergegeven, bijvoorbeeld bij een presentatie met alleen een scheidingslijn, valt de hoofdtekst terug op `---`.

Niet-ondersteunde clients blijven de terugvaltekst weergeven. Clients die OpenClaw ondersteunen, kunnen voor de weergave de voorkeur geven aan de gestructureerde metadata, terwijl ze de terugvaltekst behouden voor kopiëren, zoeken, meldingen en toegankelijkheid.

## Ondersteunde blokken

De uitgaande Matrix-adapter biedt systeemeigen ondersteuning voor:

- `buttons`
- `select`
- `context`
- `divider`

`text`-blokken worden altijd ondersteund via de terugvaltekst. Behandel alle blokken als presentatieaanwijzingen op basis van beste inspanning; negeer onbekende velden en bloktypen in plaats van het hele bericht te laten mislukken.

## Interacties

Deze metadata voegt geen Matrix-callbacksemantiek toe. Waarden van knoppen en selecties zijn terugvalpayloads voor interacties, meestal slash-opdrachten of tekstopdrachten. Een Matrix-client die interactie wil ondersteunen, bepaalt de waarde van het besturingselement (`action.command`, vervolgens `action.value`, vervolgens `value`) en stuurt die als een normaal bericht terug naar de ruimte.

Een knop met de waarde `/model deepseek/deepseek-chat` kan bijvoorbeeld worden verwerkt door die waarde als versleuteld Matrix-tekstbericht in dezelfde ruimte te verzenden.

## Relatie met goedkeuringsmetadata

`com.openclaw.presentation` is bedoeld voor algemene, uitgebreide berichtpresentatie.

Goedkeuringsprompts gebruiken de specifieke `com.openclaw.approval`-metadata, omdat goedkeuringen veiligheidsgevoelige status, beslissingen en uitvoerings-/plugingegevens bevatten. Als beide metadatasleutels in dezelfde gebeurtenis aanwezig zijn, moeten clients de voorkeur geven aan de specifieke goedkeuringsrenderer.

## Mediaberichten

Wanneer een antwoord meerdere media-URL's bevat, verzendt OpenClaw één Matrix-gebeurtenis per media-URL. Bijschrifttekst en presentatiemetadata worden alleen aan de eerste gebeurtenis toegevoegd, zodat clients één stabiele gestructureerde payload krijgen zonder dubbele renderers. Dezelfde regel geldt wanneer lange tekst over meerdere gebeurtenissen wordt verdeeld: de metadata wordt alleen met de eerste gebeurtenis meegestuurd.

Houd presentatiemetadata compact. Grote, voor gebruikers zichtbare tekst moet in `body` blijven en het normale pad voor het opsplitsen van Matrix-tekst gebruiken.
