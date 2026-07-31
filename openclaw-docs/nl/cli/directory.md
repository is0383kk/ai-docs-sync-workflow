---
read_when:
    - Je wilt contact-, groeps- of eigen ID's voor een kanaal opzoeken
    - Je ontwikkelt een adapter voor een kanaalmap
summary: CLI-referentie voor `openclaw directory` (zelf, peers, groepen)
title: Map
x-i18n:
    generated_at: "2026-07-27T05:40:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33f1cabd0954f2e6e6affbfbff9f8e1f543bffebc54baff7c1ffaa21778744a0
    source_path: cli/directory.md
    workflow: 16
---

# `openclaw directory`

Directoryzoekopdrachten voor kanalen die deze ondersteunen: contacten/peers, groepen en "me" (jezelf).

De resultaten zijn bedoeld om in andere opdrachten te plakken, met name `openclaw message send --target ...`.

## Algemene vlaggen

- `--channel <name>`: kanaal-id/-alias (vereist wanneer meerdere kanalen zijn geconfigureerd; automatisch geselecteerd wanneer er slechts één is geconfigureerd)
- `--account <id>`: account-id (standaard: standaardaccount van het kanaal)
- `--json`: uitvoer als JSON

De standaarduitvoer (niet-JSON) bestaat uit `id` (en soms `name`), gescheiden door een tab.

## Opmerkingen

- Voor veel kanalen zijn de resultaten gebaseerd op de configuratie (toegestane lijsten/geconfigureerde groepen) en niet op een live providerdirectory.
- De WhatsApp-groepslijst is live. Gateway-zoekopdrachten hergebruiken de bijbehorende beheerde verbinding; een zelfstandige opdracht opent de gekoppelde sessie alleen wanneer geen ander proces eigenaar is van dat account en meldt anders dat live groepen niet beschikbaar zijn.
- Een reeds geïnstalleerde kanaalplugin ondersteunt mogelijk geen directory's. In dat geval meldt de opdracht dat de bewerking niet wordt ondersteund; de Plugin wordt niet opnieuw geïnstalleerd of bijgewerkt om ondersteuning toe te voegen.

## Resultaten gebruiken met `message send`

```bash
openclaw directory peers list --channel slack --query "U0"
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"
```

## ID-indelingen per kanaal

| Kanaal                              | Indeling van doel-id                                                                                                        |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| WhatsApp                            | `+15551234567` (privébericht), `1234567890-1234567890@g.us` (groep), `120363123456789@newsletter` (kanaal/nieuwsbrief, alleen uitgaand) |
| Signal                              | Geconfigureerde aliassen worden omgezet in E.164-/UUID-doelen voor privéberichten of `group:<id>`-groepsdoelen |
| Telegram                            | `@username` of numerieke chat-id; groepen gebruiken numerieke id's |
| Slack                               | `user:U…` en `channel:C…` |
| Discord                             | `user:<id>` en `channel:<id>` |
| Matrix (Plugin)                     | `user:@user:server`, `room:!roomId:server` of `#alias:server` |
| Microsoft Teams (Plugin)            | `user:<id>` en `conversation:<id>` |
| Zalo (Plugin)                       | Gebruikers-id (Bot API) |
| Zalo Personal / `zalouser` (Plugin) | Thread-id (privébericht/groep), uit `zca` (`me`, `friend list`, `group list`) |

## Jezelf ("me")

```bash
openclaw directory self --channel zalouser
```

## Peers (contacten/gebruikers)

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

## Groepen

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```

## Gerelateerd

- [CLI-referentie](/nl/cli)
