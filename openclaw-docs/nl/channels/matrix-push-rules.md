---
read_when:
    - Stille streaming voor Matrix instellen voor zelfgehoste Synapse of Tuwunel
    - Gebruikers willen alleen meldingen voor voltooide blokken, niet bij elke bewerking van het voorbeeld
summary: Matrix-pushregels per ontvanger voor stille definitieve previewbewerkingen
title: Matrix-pushregels voor stille voorvertoningen
x-i18n:
    generated_at: "2026-07-27T05:24:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c58e7e796c3ae6d1ee25de229e4592ab8b4fb4d0d50a9cf868ab5ef35b1dab5
    source_path: channels/matrix-push-rules.md
    workflow: 16
---

Wanneer `channels.matrix.streaming.mode` `"quiet"` is, streamt OpenClaw het antwoord door één voorbeeldgebeurtenis ter plaatse te bewerken. Voorbeelden worden als niet-meldende `m.notice`-gebeurtenissen verzonden en de voltooide bewerking wordt gemarkeerd met `content["com.openclaw.finalized_preview"] = true`. Matrix-clients sturen bij die definitieve bewerking alleen een melding als een pushregel per gebruiker overeenkomt met de markering. Deze pagina is bedoeld voor beheerders die Matrix zelf hosten en die regel voor elk ontvangend account willen installeren.

`streaming.mode: "progress"` voltooit concepten via hetzelfde pad, zodat dezelfde regel ook wordt geactiveerd voor voltooide bewerkingen in de voortgangsmodus.

Als je alleen het standaardmeldingsgedrag van Matrix wilt, gebruik je `streaming.mode: "partial"` of laat je streaming uitgeschakeld. Zie [Matrix-kanaal instellen](/nl/channels/matrix#streaming-previews).

## Vereisten

- ontvangende gebruiker = de persoon die de melding moet ontvangen
- botgebruiker = het OpenClaw Matrix-account dat het antwoord verzendt
- gebruik voor de onderstaande API-aanroepen het toegangstoken van de ontvangende gebruiker
- laat `sender` in de pushregel overeenkomen met de volledige MXID van de botgebruiker
- het ontvangende account moet al werkende pushers hebben; regels voor stille voorbeelden werken alleen als de normale pushbezorging van Matrix goed functioneert

## Stappen

<Steps>
  <Step title="Stille voorbeelden configureren">

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "quiet" },
    },
  },
}
```

  </Step>

  <Step title="Het toegangstoken van de ontvanger ophalen">
    Gebruik waar mogelijk het token van een bestaande clientsessie opnieuw. Zo maak je een nieuw token:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

  </Step>

  <Step title="Controleren of er pushers bestaan">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Als er geen pushers worden geretourneerd, herstel je eerst de normale pushbezorging van Matrix voor dit account voordat je doorgaat.

  </Step>

  <Step title="De overschrijvende pushregel installeren">
    Installeer een regel die overeenkomt met de markering voor het voltooide voorbeeld en met de MXID van de bot als afzender:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    Vervang vóór het uitvoeren:

    - `https://matrix.example.org`: de basis-URL van je homeserver
    - `$USER_ACCESS_TOKEN`: het toegangstoken van de ontvangende gebruiker
    - `openclaw-finalized-preview-botname`: een regel-ID die per bot en per ontvanger uniek is (patroon: `openclaw-finalized-preview-<botname>`)
    - `@bot:example.org`: de MXID van je OpenClaw-bot, niet die van de ontvanger

  </Step>

  <Step title="Controleren">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Test vervolgens een gestreamd antwoord. In de stille modus toont de ruimte een stil conceptvoorbeeld en wordt er een melding gestuurd zodra het blok of de beurt is voltooid.

  </Step>
</Steps>

Om de regel later te verwijderen, `DELETE` je dezelfde regel-URL met het token van de ontvanger.

## Opmerkingen voor meerdere bots

Pushregels worden geïndexeerd op `ruleId`: als je `PUT` opnieuw uitvoert voor dezelfde ID, wordt één regel bijgewerkt. Als meerdere OpenClaw-bots dezelfde ontvanger meldingen sturen, maak je per bot één regel met een afzonderlijke overeenkomst voor de afzender.

Nieuwe door gebruikers gedefinieerde `override`-regels worden vóór de standaardonderdrukkingsregels van de server ingevoegd, zodat er geen extra parameter voor de volgorde nodig is. De regel heeft alleen invloed op tekstuele voorbeeldbewerkingen die ter plaatse kunnen worden voltooid; media-antwoorden, terugvallen voor verouderde voorbeelden en definitieve teksten die Matrix-vermeldingen zouden activeren, worden in plaats daarvan als normale berichten met meldingen bezorgd.

## Opmerkingen voor homeservers

<AccordionGroup>
  <Accordion title="Synapse">
    Er is geen speciale wijziging aan `homeserver.yaml` vereist. Als normale Matrix-meldingen deze gebruiker al bereiken, vormen het token van de ontvanger en de bovenstaande `pushrules`-aanroep de belangrijkste installatiestap.

    Als je Synapse achter een reverse proxy of met workers uitvoert, zorg er dan voor dat `/_matrix/client/.../pushrules/` Synapse correct bereikt. Pushbezorging wordt afgehandeld door het hoofdproces of `synapse.app.pusher` / geconfigureerde pusher-workers — zorg dat deze goed functioneren.

    De regel gebruikt de pushregelvoorwaarde `event_property_is` (MSC3758, pushregel v1.10), die in 2023 aan Synapse is toegevoegd. Oudere Synapse-versies accepteren de `PUT pushrules/...`-aanroep, maar voldoen ongemerkt nooit aan de voorwaarde — werk Synapse bij als er geen melding binnenkomt bij een voltooide voorbeeldbewerking.

  </Accordion>

  <Accordion title="Tuwunel">
    Dezelfde procedure als voor Synapse; er is geen Tuwunel-specifieke configuratie nodig voor de markering van het voltooide voorbeeld.

    Als meldingen verdwijnen terwijl de gebruiker actief is op een ander apparaat, controleer je of `suppress_push_when_active` is ingeschakeld. Tuwunel heeft deze optie toegevoegd in 1.4.2 (september 2025) en kan hiermee opzettelijk pushmeldingen naar andere apparaten onderdrukken zolang één apparaat actief is.

  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Matrix-kanaal instellen](/nl/channels/matrix)
- [Streamingconcepten](/nl/concepts/streaming)
