---
read_when:
    - Altijd actieve groeps- of kanaalruimten configureren
    - Je wilt dat de agent de gesprekken in de ruimte volgt zonder automatisch definitieve tekst te plaatsen
    - Fouten opsporen in typindicatie en tokengebruik zonder zichtbaar bericht in de ruimte
sidebarTitle: Ambient room events
summary: Laat ondersteunde groepsruimtes stille context bieden, tenzij de agent via de berichtentool verzendt
title: Omgevingsgebeurtenissen in ruimtes
x-i18n:
    generated_at: "2026-07-27T05:42:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 15c083c139058c9bd2c651794965bd8252d74691e536db2ad2a2ae0b4ac886e8
    source_path: channels/ambient-room-events.md
    workflow: 16
---

Omgevingsgebeurtenissen in ruimtes stellen OpenClaw in staat om niet-vermelde gesprekken in groepen of kanalen als stille context te verwerken. De agent kan het geheugen en de sessiestatus bijwerken, maar de ruimte blijft stil tenzij de agent expliciet de tool `message` aanroept.

Combineer voor groepschats die altijd actief zijn `messages.groupChat.unmentionedInbound: "room_event"` met `messages.groupChat.visibleReplies: "message_tool"`. De agent luistert, beslist wanneer een antwoord nuttig is en heeft nooit het oude promptpatroon nodig waarbij met `NO_REPLY` wordt geantwoord.

Momenteel ondersteund: Discord-gildekanalen, Slack-kanalen en privékanalen, Slack-DM's met meerdere personen en Telegram-groepen of -supergroepen. Andere groepskanalen behouden hun bestaande groepsgedrag, tenzij op hun kanaalpagina staat dat ze omgevingsgebeurtenissen in ruimtes ondersteunen.

## Aanbevolen configuratie

Stel het algemene gedrag voor groepschats in:

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
}
```

Maak de ruimte vervolgens altijd actief door de vermeldingsvereiste voor die ruimte uit te schakelen. De ruimte moet nog steeds voldoen aan de normale `groupPolicy`, de toelatingslijst voor ruimtes en de toelatingslijst voor afzenders.

Na het opslaan van de configuratie past de Gateway de instellingen voor `messages` direct toe. Start alleen opnieuw wanneer bestandsbewaking of het opnieuw laden van de configuratie is uitgeschakeld (`gateway.reload.mode: "off"`).

## Wat verandert

Met `messages.groupChat.unmentionedInbound: "room_event"`:

- toegestane groeps- of kanaalberichten zonder vermelding worden stille ruimtegebeurtenissen
- berichten met een vermelding blijven gebruikersverzoeken
- tekstuele besturingsopdrachten en systeemeigen opdrachten blijven gebruikersverzoeken
- verzoeken om af te breken of te stoppen blijven gebruikersverzoeken
- directe berichten blijven gebruikersverzoeken

Ruimtegebeurtenissen gebruiken strikte zichtbare aflevering. De uiteindelijke tekst van de assistent is privé. De agent moet `message(action=send)` aanroepen om iets in de ruimte te plaatsen.

Typindicaties en statusreacties voor de levenscyclus blijven onderdrukt voor ruimtegebeurtenissen. De enige expliciete uitzondering voor ontvangstbevestigingen is `messages.ackReactionScope: "all"`, waarmee de geconfigureerde bevestigingsreactie wordt verzonden; gebruik een beperktere reikwijdte of `"off"` wanneer de ruimte volledig stil moet blijven.

## Discord-voorbeeld

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          requireMention: false,
          users: ["<YOUR_DISCORD_USER_ID>"],
        },
      },
    },
  },
}
```

Gebruik Discord-configuratie per kanaal wanneer slechts één kanaal als omgevingscontext moet dienen. Onder `groupPolicy: "allowlist"` wordt het kanaal toegestaan door het op te nemen (`enabled: false` schakelt een vermelding uit):

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          channels: {
            "<DISCORD_CHANNEL_ID_OR_NAME>": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

## Slack-voorbeeld

Toelatingslijsten voor Slack-kanalen werken primair met ID's. Gebruik kanaal-ID's zoals `C12345678`, niet `#channel-name`. Het kanaal wordt toegestaan door het onder `channels.slack.channels` op te nemen (`enabled: false` schakelt een vermelding uit):

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    slack: {
      groupPolicy: "allowlist",
      channels: {
        "<SLACK_CHANNEL_ID>": {
          requireMention: false,
        },
      },
    },
  },
}
```

## Telegram-voorbeeld

Voor Telegram-groepen moet de bot normale groepsberichten kunnen zien. Als `requireMention: false`, schakel je de privacymodus van BotFather uit of gebruik je een andere Telegram-configuratie die al het groepsverkeer aan de bot doorgeeft.

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    telegram: {
      groups: {
        "<TELEGRAM_GROUP_CHAT_ID>": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

Telegram-groeps-ID's zijn meestal negatieve getallen, zoals `-1001234567890`. Lees `chat.id` uit `openclaw logs --follow`, stuur een groepsbericht door naar een bot die ID's opzoekt of inspecteer `getUpdates` van de Bot API.

## Agentspecifiek beleid

Gebruik een agentoverschrijving wanneer meerdere agents dezelfde ruimte delen, maar slechts één niet-vermelde gesprekken als omgevingscontext moet behandelen:

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          unmentionedInbound: "room_event",
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
}
```

De agentspecifieke waarde `agents.entries.*.groupChat.unmentionedInbound` overschrijft `messages.groupChat.unmentionedInbound` voor die agent.

## Modi voor zichtbare antwoorden

`messages.groupChat.visibleReplies` is standaard ingesteld op `"automatic"` voor normale gebruikersverzoeken in groepen en kanalen. Behoud die standaard wanneer de uiteindelijke tekst van de assistent zichtbaar moet worden geplaatst zonder expliciete aanroep van de berichtentool.

Voor altijd actieve omgevingsruimtes blijft `messages.groupChat.visibleReplies: "message_tool"` aanbevolen, vooral met modellen van de nieuwste generatie die betrouwbaar tools gebruiken, zoals GPT-5.6 Sol. Hiermee kan de agent beslissen wanneer hij iets zegt door de berichtentool aan te roepen. Als het model uiteindelijke tekst retourneert zonder de tool aan te roepen, houdt OpenClaw die uiteindelijke tekst privé en registreert het metagegevens over de onderdrukte aflevering.

Ruimtegebeurtenissen blijven strikt, zelfs wanneer andere groepsverzoeken automatische antwoorden gebruiken. Niet-vermelde omgevingsgebeurtenissen in ruimtes vereisen altijd `message(action=send)` voor zichtbare uitvoer.

## Geschiedenis

`messages.groupChat.historyLimit` stelt de algemene standaard voor groepsgeschiedenis in (50 wanneer niet ingesteld; moet een positief geheel getal zijn). Kanalen kunnen deze overschrijven met `channels.<channel>.historyLimit`, en sommige kanalen ondersteunen ook geschiedenislimieten per account. Stel `historyLimit: 0` op kanaalniveau in om context uit de groepsgeschiedenis voor dat kanaal uit te schakelen.

Ondersteunde kanalen voor ruimtegebeurtenissen behouden recente omgevingsberichten in ruimtes als context. Telegram behoudt een altijd actief, voortschrijdend venster per groep dat wordt begrensd door `historyLimit`; beurten met gebruikersverzoeken selecteren vermeldingen na het laatst geregistreerde antwoord van de bot, terwijl beurten met ruimtegebeurtenissen het volledige recente venster ontvangen, zodat het model zijn eigen recente berichten kan zien. De buiten gebruik gestelde Telegram-modussleutel `includeGroupHistoryContext` wordt verwijderd door `openclaw doctor --fix`.

## Problemen oplossen

Als de ruimte een typindicatie of tokengebruik toont, maar geen zichtbaar bericht:

1. Controleer of de ruimte is toegestaan door de toelatingslijst voor kanalen en de toelatingslijst voor afzenders.
2. Controleer of `requireMention: false` is ingesteld op het verwachte ruimteniveau.
3. Controleer of `messages.groupChat.unmentionedInbound` of de agentoverschrijving is ingesteld op `"room_event"`.
4. Controleer de logboeken op metagegevens van onderdrukte uiteindelijke payloads of `didSendViaMessagingTool: false`.
5. Behoud of herstel voor normale groepsverzoeken `messages.groupChat.visibleReplies: "automatic"` als je wilt dat uiteindelijke antwoorden automatisch worden geplaatst. Gebruik voor omgevingsruimtes met `message_tool` een model/runtime dat betrouwbaar tools aanroept.

Als Telegram-omgevingsruimtes helemaal niet worden geactiveerd, controleer dan de privacymodus van BotFather en verifieer dat de Gateway normale groepsberichten ontvangt.

Als Slack-omgevingsruimtes niet worden geactiveerd, controleer dan of de kanaalsleutel het Slack-kanaal-ID is en of de app het geschiedenisbereik voor dat ruimtetype heeft: `channels:history` (openbaar), `groups:history` (privé) of `mpim:history` (DM's met meerdere personen).

## Gerelateerd

- [Groepen](/nl/channels/groups)
- [Discord](/nl/channels/discord)
- [Slack](/nl/channels/slack)
- [Telegram](/nl/channels/telegram)
- [Problemen met kanalen oplossen](/nl/channels/troubleshooting)
- [Referentie voor kanaalconfiguratie](/nl/gateway/config-channels)
