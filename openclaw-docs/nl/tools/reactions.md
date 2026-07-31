---
read_when:
    - Werken met reacties in elk kanaal
    - Begrijpen hoe emoji-reacties per platform verschillen
summary: Semantiek van de reactietool in alle ondersteunde kanalen
title: Reacties
x-i18n:
    generated_at: "2026-07-27T05:38:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e148a93edbcfbe997075f6e9e191667ec257f76fa48162688fd1f333479661f0
    source_path: tools/reactions.md
    workflow: 16
---

De agent voegt emoji-reacties toe en verwijdert ze met de actie `react`
van de tool `message`. Het gedrag verschilt per kanaal.

## Hoe het werkt

```json
{
  "action": "react",
  "messageId": "msg-123",
  "emoji": "thumbsup"
}
```

- `emoji` is vereist wanneer je een reactie toevoegt.
- Stel `emoji` in op een lege tekenreeks (`""`) om de reactie(s) van de bot te verwijderen op
  kanalen die dit ondersteunen.
- Stel `remove: true` in om één specifieke emoji te verwijderen (vereist een niet-lege
  `emoji`).
- Op kanalen met statusreacties zorgt `trackToolCalls: true` bij een reactie ervoor dat
  de runtime dat bericht met reactie opnieuw kan gebruiken voor latere reacties over de toolvoortgang
  tijdens dezelfde beurt.

## Gedrag per kanaal

<AccordionGroup>
  <Accordion title="Discord en Slack">
    - Een lege `emoji` verwijdert alle reacties van de bot op het bericht.
    - `remove: true` verwijdert alleen de opgegeven emoji.

  </Accordion>

  <Accordion title="Nextcloud Talk">
    - Alleen reacties toevoegen: `emoji` is vereist en mag niet leeg zijn.
    - Het verwijderen van reacties is nog niet gekoppeld aan een verwijderaanroep; `remove: true` wordt in plaats van stil niets te doen met een expliciete fout afgewezen.
    - Vereist dat de Talk-bot is geregistreerd met de functie `reaction` (zie [documentatie voor het Nextcloud Talk-kanaal](/nl/channels/nextcloud-talk)).

  </Accordion>

  <Accordion title="Telegram">
    - Een lege `emoji` verwijdert de reacties van de bot.
    - `remove: true` verwijdert eveneens reacties, maar vereist voor toolvalidatie nog steeds een niet-lege `emoji`.

  </Accordion>

  <Accordion title="WhatsApp">
    - Een lege `emoji` verwijdert de reactie van de bot.
    - `remove: true` wordt intern omgezet naar een lege emoji (vereist nog steeds `emoji` in de toolaanroep).
    - WhatsApp heeft per bericht één reactievak voor de bot; als je een nieuwe reactie verzendt, vervangt die de bestaande reactie in plaats van meerdere emoji te stapelen.

  </Accordion>

  <Accordion title="Zalo Personal (zalouser)">
    - Vereist een niet-lege `emoji` voor zowel toevoegen als verwijderen.
    - `remove: true` verwijdert die specifieke emoji-reactie.

  </Accordion>

  <Accordion title="Feishu/Lark">
    - Gebruikt dezelfde actie `react` als andere kanalen (toevoegen/verwijderen/weergeven via berichtreactie-ID's), niet een afzonderlijke tool.
    - Voor toevoegen is een niet-lege `emoji` vereist (omgezet naar een Feishu-`emoji_type`, bijvoorbeeld `SMILE`, `THUMBSUP`, `HEART`).
    - `remove: true` vereist een niet-lege `emoji` en verwijdert de eigen reactie van de bot die overeenkomt met dat emojitype.
    - Een lege `emoji` met `clearAll: true` verwijdert alle reacties van de bot op het bericht.

  </Accordion>

  <Accordion title="Signal">
    - Meldingen van inkomende reacties worden beheerd door `channels.signal.reactionNotifications`: `"off"` schakelt ze uit, `"own"` (standaard) genereert gebeurtenissen wanneer gebruikers reageren op botberichten, `"all"` genereert gebeurtenissen voor alle reacties en `"allowlist"` genereert alleen gebeurtenissen voor afzenders in `channels.signal.reactionAllowlist`.

  </Accordion>

  <Accordion title="iMessage">
    - Uitgaande reacties zijn iMessage-tapbacks (`love`, `like`, `dislike`, `laugh`, `emphasize` en `question`); `emoji` moet naar een van deze typen worden omgezet om een reactie toe te voegen.
    - `remove: true` zonder een herkend tapbacktype verwijdert alle tapbacktypen; met een herkend type verwijdert het alleen dat ene type.

  </Accordion>
</AccordionGroup>

## Reactieniveau

De `reactionLevel` per kanaal beperkt hoe vaak de agent zijn eigen
reacties verzendt. Waarden: `off`, `ack`, `minimal` of `extensive`.

- [Meldingen van Telegram-reacties](/nl/channels/telegram#feature-reference) - `channels.telegram.reactionLevel` (standaard `minimal`)
- [Reactieniveau van WhatsApp](/nl/channels/whatsapp#reaction-level) - `channels.whatsapp.reactionLevel` (standaard `minimal`)
- [Signal-reacties](/nl/channels/signal#reactions-message-tool) - `channels.signal.reactionLevel` (standaard `minimal`)

## Gerelateerd

- [Agent Send](/nl/tools/agent-send) - de tool `message` die `react` bevat
- [Kanalen](/nl/channels) - kanaalspecifieke configuratie
