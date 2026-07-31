---
read_when:
    - Botgeschreven kanaalberichten configureren
    - Bescherming tegen bot-naar-bot-lussen afstellen
sidebarTitle: Bot loop protection
summary: Standaardinstellingen voor bescherming tegen bot-naar-bot-lussen en kanaaloverschrijvingen
title: Bescherming tegen botlussen
x-i18n:
    generated_at: "2026-07-27T06:03:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d59d3b48dd5506e774282b880334df8970b05c4d001261ff7107e8e1678894db
    source_path: channels/bot-loop-protection.md
    workflow: 16
---

OpenClaw kan berichten accepteren die door andere bots zijn geschreven op kanalen die `allowBots` ondersteunen. Wanneer dat pad is ingeschakeld, voorkomt lusbeveiliging voor paren dat twee botidentiteiten elkaar eindeloos blijven antwoorden.

De beveiliging wordt afgedwongen door de centrale runner voor inkomende antwoorden. Elk ondersteunend kanaal zet zijn inkomende gebeurtenis om in generieke gegevens: account of bereik, gespreks-id, bot-id van de afzender en bot-id van de ontvanger. De kern houdt het deelnemerspaar in beide richtingen bij (A naar B en B naar A gelden als hetzelfde paar), past een budget met een schuivend venster toe en onderdrukt het paar gedurende een afkoelperiode nadat het budget is overschreden.

## Standaardwaarden

Lusbeveiliging voor paren is actief wanneer een kanaal toestaat dat door bots geschreven berichten de dispatch bereiken. Ingebouwde standaardwaarden:

| Sleutel              | Standaard | Betekenis                                           |
| -------------------- | --------- | --------------------------------------------------- |
| `enabled`            | `true`  | Beveiliging actief voor kanalen die deze ondersteunen. |
| `maxEventsPerWindow` | `20`    | Gebeurtenissen die een botpaar binnen het venster kan uitwisselen. |
| `windowSeconds`      | `60`    | Lengte van het schuivende venster.                  |
| `cooldownSeconds`    | `60`    | Onderdrukkingstijd nadat het paar het budget overschrijdt. |

De beveiliging heeft geen invloed op door mensen geschreven berichten, implementaties met één bot, het filteren van berichten aan zichzelf of botantwoorden die binnen het budget blijven.

## Gedeelde standaardwaarden configureren

Stel `channels.defaults.botLoopProtection` eenmaal in om elk ondersteunend kanaal dezelfde basisinstellingen te geven. Kanalen kunnen ook specifiekere overschrijvingen aanbieden; Feishu gebruikt bewust alleen deze gedeelde basisinstellingen.

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
  },
}
```

Stel `enabled: false` alleen in wanneer je kanaalbeleid bot-naar-botgesprekken bewust toestaat zonder automatische onderdrukking.

## Overschrijven per kanaal, account of ruimte

Ondersteunende kanalen leggen hun eigen configuratie sleutel voor sleutel over de gedeelde standaardwaarde heen. Voorrangsvolgorde, van specifiek naar algemeen:

1. `channels.<channel>.<room-or-space>.botLoopProtection`, wanneer het kanaal overschrijvingen per gesprek ondersteunt
2. `channels.<channel>.accounts.<account>.botLoopProtection`, wanneer het kanaal accounts ondersteunt
3. `channels.<channel>.botLoopProtection`, wanneer het kanaal standaardwaarden op het hoogste niveau ondersteunt
4. `channels.defaults.botLoopProtection`
5. ingebouwde standaardwaarden

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
      },
    },
    discord: {
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
      accounts: {
        secondary: {
          allowBots: true,
          botLoopProtection: {
            maxEventsPerWindow: 5,
            cooldownSeconds: 90,
          },
        },
      },
    },
    googlechat: {
      allowBots: true,
      groups: {
        "spaces/AAAA": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    matrix: {
      allowBots: "mentions",
      groups: {
        "!roomid:example.org": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    slack: {
      allowBots: "mentions",
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
    },
  },
}
```

## Kanaalondersteuning

- Discord: native `author.bot`-gegevens, geïndexeerd op Discord-account, kanaal en botpaar.
- Feishu: native `sender_type=bot`-gegevens voor toegelaten, door bots geschreven groepsberichten, geïndexeerd op Feishu-account, chat en botpaar. Feishu gebruikt alleen `channels.defaults.botLoopProtection`.
- Google Chat: native `sender.type=BOT`-gegevens voor geaccepteerde, door bots geschreven berichten, geïndexeerd op account, ruimte en botpaar.
- Matrix: geconfigureerde Matrix-botaccounts, geïndexeerd op Matrix-account, ruimte en geconfigureerd botpaar.
- Slack: native `bot_id`-gegevens voor geaccepteerde, door bots geschreven berichten, geïndexeerd op Slack-account, kanaal en botpaar.

Kanalen die geen betrouwbare identiteit van een inkomende bot beschikbaar stellen, blijven hun normale filters voor berichten aan zichzelf en toegangsbeleid gebruiken. Ze moeten deze beveiliging pas inschakelen wanneer ze beide deelnemers van het botpaar kunnen identificeren.

Zie [SDK-runtime](/nl/plugins/sdk-runtime#reusable-runtime-utilities) voor implementatiedetails voor plugins.
