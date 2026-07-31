---
read_when:
    - Je wilt een Feishu/Lark-bot verbinden
    - Je configureert het Feishu-kanaal
summary: Overzicht, functies en configuratie van de Feishu-bot
title: Feishu
x-i18n:
    generated_at: "2026-07-27T05:42:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e7c4cbb704ce266b7c7b0f6e160c36c873050fee8d5808965e15b56ad637f28
    source_path: channels/feishu.md
    workflow: 16
---

OpenClaw maakt verbinding met Feishu/Lark (het alles-in-één-samenwerkingsplatform) via de officiële `@openclaw/feishu`-plugin: privéberichten met de bot, groepschats, streamende kaartantwoorden en tools voor Feishu-documenten, wiki's, Drive en Bitable.

**Status:** productieklaar voor privéberichten met de bot en groepschats. WebSocket is het standaardeventtransport (geen openbare URL nodig); de webhookmodus is optioneel.

## Snel aan de slag

<Note>
Vereist OpenClaw 2026.5.29 of hoger. Voer `openclaw --version` uit om dit te controleren. Upgrade met `openclaw update`.
</Note>

<Steps>
  <Step title="Voer de configuratiewizard voor het kanaal uit">
  ```bash
  openclaw channels login --channel feishu
  ```
  Hiermee wordt de `@openclaw/feishu`-plugin geïnstalleerd als die ontbreekt, waarna je door de configuratie wordt geleid:

- **Handmatige configuratie**: plak een App ID en App Secret van Feishu Open Platform (`https://open.feishu.cn`) of Lark Developer (`https://open.larksuite.com`).
- **QR-configuratie**: scan een QR-code in de Feishu-app om automatisch een bot te maken. Deze procedure beperkt privéberichten tot je eigen account (`dmPolicy: "allowlist"` met je `open_id`).

De wizard vraagt ook naar het API-domein (Feishu of Lark) en het groepsbeleid. Als de binnenlandse mobiele Feishu-app niet op de QR-code reageert, voer je de configuratie opnieuw uit en kies je handmatige configuratie.
</Step>

  <Step title="Start nadat de configuratie is voltooid de Gateway opnieuw om de wijzigingen toe te passen">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

## Duurzaamheid van inkomende events

OpenClaw plaatst geverifieerde `im.message.receive_v1`- en `drive.notice.comment_add_v1`-enveloppen duurzaam in de wachtrij voordat ze naar de agent worden doorgestuurd. Openstaande events of events die opnieuw kunnen worden geprobeerd, overleven een herstart van de Gateway, blijven per chat of document geserialiseerd en gebruiken de event-ID van Feishu om dubbele wachtrij-items te onderdrukken zolang de actieve of bewaarde voltooiingsregistratie bestaat.

Als een WebSocket-event na een begrensd aantal pogingen niet kan worden opgeslagen, sluit OpenClaw die socket en dwingt het een nieuwe geverifieerde verbinding af in plaats van door te gaan na een niet-vastgelegde beurt. Andere Feishu-eventtypen, waaronder reacties en uitnodigingen voor VC-vergaderingen, gebruiken hun normale eventpaden en vallen niet onder deze garantie voor duurzame wachtrijen.

## Toegangsbeheer

### Privéberichten

Configureer `channels.feishu.dmPolicy` (standaard: `pairing`) om te bepalen wie privéberichten naar de bot kan sturen:

| Waarde         | Gedrag                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| `"pairing"`   | Onbekende gebruikers ontvangen een koppelingscode; keur deze goed via de CLI                                                         |
| `"allowlist"` | Alleen gebruikers die in `allowFrom` staan, kunnen chatten                                                                     |
| `"open"`      | Openbare privéberichten; configuratievalidatie vereist dat `allowFrom` `"*"` bevat. Items zonder jokerteken beperken de toegang nog steeds |

**Een koppelingsverzoek goedkeuren:**

```bash
openclaw pairing list feishu
openclaw pairing approve feishu <CODE>
```

### Groepschats

**Groepsbeleid** (`channels.feishu.groupPolicy`, standaard: `allowlist`):

| Waarde         | Gedrag                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `"open"`      | Reageren op alle berichten in groepen                                                            |
| `"allowlist"` | Alleen reageren op groepen in `groupAllowFrom` of groepen die expliciet onder `groups.<chat_id>` zijn geconfigureerd |
| `"disabled"`  | Alle groepsberichten uitschakelen; expliciete `groups.<chat_id>`-items overschrijven dit niet         |

**Vermeldingsvereiste** (`channels.feishu.requireMention`):

- Standaard is een @vermelding vereist, behalve wanneer het effectieve groepsbeleid `"open"` is; dan is de standaardwaarde `false`, zodat berichten die geen vermeldingen kunnen bevatten (bijvoorbeeld afbeeldingen) de agent toch bereiken.
- Stel `true` of `false` expliciet in om dit te overschrijven; overschrijving per groep: `channels.feishu.groups.<chat_id>.requireMention`.
- De uitsluitend voor uitzending bestemde `@all` en `@_all` worden niet als botvermeldingen beschouwd. Een bericht waarin zowel `@all` als de bot rechtstreeks wordt vermeld, telt nog steeds als een botvermelding.

## Voorbeelden van groepsconfiguratie

### Alle groepen toestaan, geen @vermelding vereist

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open", // requireMention is standaard false bij "open"
    },
  },
}
```

### Alle groepen toestaan, maar nog steeds een @vermelding vereisen

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true,
    },
  },
}
```

### Alleen specifieke groepen toestaan

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      // Groeps-ID's zien eruit als: oc_xxx
      groupAllowFrom: ["oc_xxx", "oc_yyy"],
    },
  },
}
```

In de modus `allowlist` kun je een groep ook toelaten door een expliciet `groups.<chat_id>`-item toe te voegen. Expliciete items overschrijven `groupPolicy: "disabled"` niet. Standaardwaarden met jokertekens onder `groups.*` configureren overeenkomende groepen, maar laten zelf geen groepen toe.

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groups: {
        oc_xxx: {
          requireMention: false,
        },
      },
    },
  },
}
```

### Afzenders binnen een groep beperken

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          // Open-ID's van gebruikers zien eruit als: ou_xxx
          allowFrom: ["ou_user1", "ou_user2"],
        },
      },
    },
  },
}
```

`channels.feishu.groupSenderAllowFrom` stelt voor alle groepen dezelfde toelatingslijst voor afzenders in; een `allowFrom` per groep heeft voorrang.

### Door bots geschreven berichten

Feishu negeert standaard berichten die door andere bots zijn geschreven. Om bot-naar-botgroepsgesprekken toe te staan, verleen je de app de scopes `im:message.group_at_msg.include_bot:readonly` en `im:message:readonly` en stel je vervolgens `allowBots` in:

```json5
{
  channels: {
    feishu: {
      allowBots: true,
    },
  },
}
```

Feishu levert door bots geschreven groepsevents alleen wanneer een andere bot deze bot vermeldt. Het bestaande groepsbeleid, de toelatingslijsten voor afzenders en de vermeldingsvereisten blijven van toepassing. OpenClaw verwijdert zelfgeschreven berichten, vermeldt de andere bot in elk tekst- of kaartantwoord en past de gedeelde beveiliging tegen [`channels.defaults.botLoopProtection`](/nl/channels/bot-loop-protection) toe.

<a id="get-groupuser-ids"></a>

## Groeps-/gebruikers-ID's ophalen

### Groeps-ID's (`chat_id`, indeling: `oc_xxx`)

Open de groep in Feishu/Lark, klik rechtsboven op het menupictogram en ga naar **Settings**. De groeps-ID (`chat_id`) wordt op de instellingenpagina vermeld.

![Groeps-ID ophalen](/images/feishu-get-group-id.png)

### Gebruikers-ID's (`open_id`, indeling: `ou_xxx`)

Start de Gateway, stuur een privébericht naar de bot en controleer vervolgens de logboeken:

```bash
openclaw logs --follow
```

Zoek naar `open_id` in de loguitvoer. Je kunt ook openstaande koppelingsverzoeken controleren:

```bash
openclaw pairing list feishu
```

## Veelgebruikte opdrachten

| Opdracht   | Beschrijving                 |
| --------- | --------------------------- |
| `/status` | Botstatus weergeven             |
| `/reset`  | De huidige sessie opnieuw instellen   |
| `/model`  | Het AI-model weergeven of wijzigen |

<Note>
Feishu/Lark ondersteunt geen systeemeigen menu's voor slashopdrachten, dus stuur deze als gewone tekstberichten.
</Note>

## Problemen oplossen

### De bot reageert niet in groepschats

1. Zorg dat de bot aan de groep is toegevoegd
2. Zorg dat je de bot @vermeldt (standaard vereist)
3. Controleer of `groupPolicy` niet `"disabled"` is
4. Controleer de logboeken: `openclaw logs --follow`

### De bot ontvangt geen berichten

1. Zorg dat de bot in Feishu Open Platform / Lark Developer is gepubliceerd en goedgekeurd
2. Zorg dat het eventabonnement `im.message.receive_v1` bevat
3. Abonneer je voor automatisch deelnemen aan vergaderuitnodigingen ook op `vc.bot.meeting_invited_v1`
4. Zorg dat **persistent connection** (WebSocket) is geselecteerd
5. Zorg dat alle vereiste machtigingsscopes zijn verleend
6. Zorg dat de Gateway actief is: `openclaw gateway status`
7. Controleer de logboeken: `openclaw logs --follow`

Een abonnement op `vc.bot.meeting_invited_v1` levert alleen het event. Automatisch deelnemen is
standaard uitgeschakeld. Om dit globaal in te schakelen:

```json5
{
  channels: {
    feishu: {
      vcAutoJoin: true,
    },
  },
}
```

Om dit voor slechts één account in te schakelen, laat je de schakelaar op het hoogste niveau weg en stel je de overschrijving voor het account in:

```json5
{
  channels: {
    feishu: {
      accounts: {
        meetings: { vcAutoJoin: true },
      },
    },
  },
}
```

Uitnodigers doorlopen nog steeds het normale Feishu-beleid voor privéberichten, de toelatingslijst/koppeling, de sessie en de routering van antwoorden
voordat de agent een beurt voor deelname ontvangt. Voor deelname is ook een beschikbare Feishu VC-deelnametool
vereist die is geconfigureerd voor de app-identiteit met de scope
`vc:meeting.bot.join:write`. De officiële
[`lark-cli` VC-agent-Skill](https://github.com/larksuite/cli/tree/main/skills/lark-vc-agent)
biedt bijvoorbeeld `vc +meeting-join`.

<Warning>
De officiële `lark-cli` VC-agent-Skill markeert acties voor vergaderbots momenteel als een beperkte bèta. Als de tool `ErrNotInGray` of foutcode `20017` retourneert, is de app of tenant niet voor die bèta ingeschakeld; gebruik de richtlijnen voor vroege toegang in de gekoppelde Skill voordat je problemen met gewone scopetoekenningen oplost.
</Warning>

### QR-configuratie reageert niet in de mobiele Feishu-app

1. Voer de configuratie opnieuw uit: `openclaw channels login --channel feishu`
2. Kies handmatige configuratie
3. Maak in Feishu Open Platform een zelfgebouwde app en kopieer de App ID en App Secret
4. Plak die aanmeldgegevens in de configuratiewizard

### App Secret is gelekt

1. Stel de App Secret opnieuw in via Feishu Open Platform / Lark Developer
2. Werk de waarde in je configuratie bij
3. Start de Gateway opnieuw: `openclaw gateway restart`

## Geavanceerde configuratie

### Meerdere accounts

```json5
{
  channels: {
    feishu: {
      defaultAccount: "main",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "Primaire bot",
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
        backup: {
          appId: "cli_yyy",
          appSecret: "yyy",
          name: "Reservebot",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` bepaalt welk account wordt gebruikt wanneer uitgaande API's geen `accountId` opgeven. Accountitems nemen instellingen op het hoogste niveau over; de meeste sleutels op het hoogste niveau kunnen per account worden overschreven.
`accounts.<id>.tts` gebruikt dezelfde structuur als `tts` en wordt diep samengevoegd over de globale TTS-configuratie, zodat Feishu-configuraties met meerdere bots gedeelde providerreferenties globaal kunnen behouden en per account alleen de stem, het model, de persona of de automatische modus hoeven te overschrijven.

### Berichtlimieten

- `textChunkLimit` - segmentgrootte voor uitgaande tekst (standaard: `4000` tekens)
- `streaming.chunkMode` - `"length"` (standaard) splitst bij de limiet; `"newline"` geeft de voorkeur aan regeleinden
- `mediaMaxMb` - limiet voor het uploaden/downloaden van media (standaard: `30` MB)

### Streaming

Feishu/Lark ondersteunt streamende antwoorden via interactieve kaarten (Card Kit-streaming-API). Wanneer dit is ingeschakeld, werkt de bot de kaart in realtime bij terwijl de tekst wordt gegenereerd.

```json5
{
  channels: {
    feishu: {
      streaming: {
        mode: "partial", // streamende kaartuitvoer (standaard: "partial")
        block: { enabled: true }, // voltooide-blokstreaming inschakelen
      },
    },
  },
}
```

Stel `streaming.mode: "off"` in om het volledige antwoord in één bericht te verzenden; `renderMode: "raw"` (platte tekst in plaats van kaarten) schakelt streamingkaarten eveneens uit. `streaming.block.enabled` is standaard uitgeschakeld; schakel dit alleen in als je voltooide assistentblokken vóór het definitieve antwoord wilt laten verzenden. De verouderde booleaanse waarde `streaming` en de platte sleutels `blockStreaming` / `blockStreamingCoalesce` / `chunkMode` worden via `openclaw doctor --fix` naar deze geneste structuur gemigreerd.

### Quotaoptimalisatie

Verminder het aantal Feishu/Lark-API-aanroepen met twee optionele vlaggen:

- `typingIndicator` (standaard `true`): stel `false` in om aanroepen voor typreacties over te slaan
- `resolveSenderNames` (standaard `true`): stel `false` in om het opzoeken van afzenderprofielen over te slaan

```json5
{
  channels: {
    feishu: {
      typingIndicator: false,
      resolveSenderNames: false,
    },
  },
}
```

### Bereik van groepssessies en onderwerpthreads

`channels.feishu.groupSessionScope` (op het hoogste niveau, per account of per groep) bepaalt hoe groepsberichten aan agentsessies worden gekoppeld:

| Waarde                  | Sessie                                                          |
| ---------------------- | ---------------------------------------------------------------- |
| `"group"` (standaard)    | Eén sessie per groepschat                                       |
| `"group_sender"`       | Eén sessie per (groep + afzender)                                 |
| `"group_topic"`        | Eén sessie per onderwerpthread; valt terug op de groepssessie    |
| `"group_topic_sender"` | Eén sessie per (onderwerp + afzender); valt terug op (groep + afzender) |

Voor de onderwerpbereiken gebruiken systeemeigen Feishu/Lark-onderwerpgroepen de gebeurtenis `thread_id` (`omt_*`) als canonieke sleutel voor de onderwerpsessie. Als bij een systeemeigen startgebeurtenis van een onderwerp `thread_id` ontbreekt, haalt OpenClaw deze vóór het routeren van de beurt op uit Feishu. Normale groepsantwoorden die OpenClaw omzet in threads, blijven de bericht-ID van het hoofdantwoord (`om_*`) gebruiken, zodat de eerste beurt en vervolgbeurten in dezelfde sessie blijven.

Stel `replyInThread: "enabled"` in (op het hoogste niveau of per groep) om botantwoorden een Feishu-onderwerpthread te laten maken of voortzetten in plaats van inline te antwoorden. `topicSessionMode` is de verouderde voorganger van `groupSessionScope`; geef de voorkeur aan `groupSessionScope`.

### Feishu-werkruimtetools

De Plugin bevat agenttools voor Feishu-documenten, chats, de kennisbank, cloudopslag, machtigingen en Bitable, plus bijbehorende Skills (`feishu-doc`, `feishu-drive`, `feishu-perm`, `feishu-wiki`). Toolfamilies worden beheerd via `channels.feishu.tools`:

| Sleutel             | Tools                                         | Standaard             |
| --------------- | --------------------------------------------- | ------------------- |
| `tools.doc`     | `feishu_doc` documentbewerkingen              | `true`              |
| `tools.chat`    | `feishu_chat` chatinformatie + ledenquery's      | `true`              |
| `tools.wiki`    | `feishu_wiki` kennisbank (vereist `doc`) | `true`              |
| `tools.drive`   | `feishu_drive` cloudopslag                  | `true`              |
| `tools.perm`    | `feishu_perm` machtigingenbeheer           | `false` (gevoelig) |
| `tools.scopes`  | `feishu_app_scopes` diagnostiek van appbereiken     | `true`              |
| `tools.bitable` | `feishu_bitable_*` Bitable/Base-bewerkingen    | `true`              |

`tools.base` is een alias voor `tools.bitable`; de expliciete waarde `bitable` heeft voorrang wanneer beide zijn ingesteld. Instellingen per account staan onder `accounts.<id>.tools`.

Verleen `drive:drive.metadata:readonly` voor rechtstreekse `feishu_drive info`-zoekopdrachten buiten de hoofdmap,
tenzij de app al het volledige bereik `drive:drive` heeft. Zonder een van beide bereiken houdt `info`
de verouderde zoekopdracht in de hoofdmap beschikbaar via `drive:drive:readonly`.

### ACP-sessies

Feishu/Lark ondersteunt ACP voor privéberichten en berichten in groepsthreads. Feishu/Lark-ACP wordt aangestuurd met tekstopdrachten — er zijn geen systeemeigen menu's voor slash-opdrachten, dus gebruik `/acp ...`-berichten rechtstreeks in het gesprek.

#### Permanente ACP-koppeling

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "direct", id: "ou_1234567890" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "group", id: "oc_group_chat:topic:om_topic_root" },
      },
      acp: { label: "codex-feishu-topic" },
    },
  ],
}
```

#### ACP starten vanuit een chat

In een Feishu/Lark-privébericht of -thread:

```text
/acp spawn codex --thread here
```

`--thread here` werkt voor privéberichten en Feishu/Lark-threadberichten. Vervolgberichten in het gekoppelde gesprek worden rechtstreeks naar die ACP-sessie gerouteerd.

### Routering met meerdere agents

Gebruik `bindings` om Feishu/Lark-privéberichten of -groepen naar verschillende agents te routeren.

```json5
{
  agents: {
    list: [
      { id: "main" },
      { id: "agent-a", workspace: "/home/user/agent-a" },
      { id: "agent-b", workspace: "/home/user/agent-b" },
    ],
  },
  bindings: [
    {
      agentId: "agent-a",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzz" },
      },
    },
  ],
}
```

Routeringsvelden:

- `match.channel`: `"feishu"`
- `match.peer.kind`: `"direct"` (privébericht) of `"group"` (groepschat)
- `match.peer.id`: Open ID van de gebruiker (`ou_xxx`) of groeps-ID (`oc_xxx`)

Zie [Groeps-/gebruikers-ID's ophalen](#get-groupuser-ids) voor zoektips.

## Agentisolatie per gebruiker (dynamische agentaanmaak)

Schakel `dynamicAgentCreation` in om automatisch **geïsoleerde agentinstanties** te maken voor elke gebruiker van privéberichten. Elke gebruiker krijgt een eigen:

- Onafhankelijke werkruimtemap
- Afzonderlijke `USER.md` / `SOUL.md` / `MEMORY.md`
- Privégespreksgeschiedenis
- Geïsoleerde Skills en status

Dit is essentieel voor openbare bots waarbij je elke gebruiker een eigen privé-ervaring met een AI-assistent wilt bieden.

<Note>
Dynamische koppelingen bevatten de genormaliseerde Feishu-`accountId`, zodat standaardaccounts en benoemde accounts elke afzender naar de juiste dynamische agent routeren.

Als een benoemd account in een oudere versie een dynamische agent zonder bereik heeft gemaakt, telt die verouderde agent nog steeds mee voor `maxAgents`. Controleer of deze niet door het standaardaccount wordt gebruikt voordat je deze verwijdert, of verhoog `maxAgents` tijdelijk; OpenClaw kan niet veilig afleiden welk account eigenaar is van een dubbelzinnige verouderde status.
</Note>

### Snelle installatie

```json5
{
  channels: {
    feishu: {
      dmPolicy: "open",
      allowFrom: ["*"],
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // Cruciaal: maakt van het privébericht van elke gebruiker diens "hoofdsessie"
    // Laadt automatisch USER.md / SOUL.md / MEMORY.md
    // Gebruik voor sterkere isolatie in plaats daarvan "per-channel-peer"
    dmScope: "main",
  },
}
```

### Werking

Wanneer een nieuwe gebruiker het eerste privébericht verzendt:

1. Het kanaal genereert een unieke `agentId`: `feishu-{user_open_id}` voor het standaardaccount, of een begrensde identiteitsdigest met accountvoorvoegsel voor een benoemd account
2. Maakt een nieuwe werkruimte op het pad `workspaceTemplate`
3. Registreert de agent en maakt een koppeling voor deze gebruiker
4. De werkruimtehelper zorgt bij de eerste toegang voor bootstrapbestanden (`AGENTS.md`, `SOUL.md`, `USER.md`, enzovoort)
5. Routeert alle toekomstige berichten van deze gebruiker naar diens toegewezen agent

### Configuratieopties

| Instelling                                                  | Beschrijving                                | Standaard                              |
| -------------------------------------------------------- | ------------------------------------------ | ------------------------------------ |
| `channels.feishu.dynamicAgentCreation.enabled`           | Automatische agentaanmaak per gebruiker inschakelen   | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | Padsjabloon voor dynamische agentwerkruimten | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | Naamsjabloon voor de agentmap              | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | Maximumaantal dynamische agents dat kan worden gemaakt | onbeperkt                            |

Sjabloonvariabelen:

- `{agentId}` - de gegenereerde agent-ID (bijvoorbeeld `feishu-ou_xxxxxx` of `feishu-support-<identity_digest>`)
- `{userId}` - de Feishu-open_id van de afzender (bijvoorbeeld `ou_xxxxxx`)

### Sessiebereik

`session.dmScope` bepaalt hoe privéberichten aan agentsessies worden gekoppeld. Dit is een **globale instelling** die alle kanalen beïnvloedt.

| Waarde                        | Gedrag                                                            | Het meest geschikt voor                                                           |
| ---------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `"main"`                     | Het privébericht van elke gebruiker wordt aan de hoofdsessie van diens agent gekoppeld                   | Bots voor één gebruiker waarbij je `USER.md` / `SOUL.md` automatisch wilt laden |
| `"per-peer"`                 | Elke gesprekspartner krijgt een afzonderlijke sessie (ongeacht het kanaal)           | Isolatie uitsluitend op basis van de identiteit van de afzender                            |
| `"per-channel-peer"`         | Elke combinatie van (kanaal + gebruiker) krijgt een afzonderlijke sessie           | Openbare bots voor meerdere gebruikers die sterkere isolatie nodig hebben                  |
| `"per-account-channel-peer"` | Elke combinatie van (account + kanaal + gebruiker) krijgt een afzonderlijke sessie | Bots met meerdere accounts die sessie-isolatie op accountniveau nodig hebben         |

**Afweging**: met `"main"` worden bootstrapbestanden automatisch geladen (`USER.md`, `SOUL.md`, `MEMORY.md`), maar gebruiken alle privéberichten in alle kanalen hetzelfde patroon voor sessiesleutels. Overweeg voor openbare bots met meerdere gebruikers, waarbij isolatie belangrijker is dan het automatisch laden van bootstrapbestanden, `"per-channel-peer"` en beheer de bootstrapbestanden handmatig.

<Note>
Gebruik `"per-account-channel-peer"` wanneer benoemde Feishu-accounts afzonderlijke sessies voor dezelfde afzender moeten behouden. Dynamische koppelingen behouden het accountbereik.
</Note>

### Gebruikelijke implementatie voor meerdere gebruikers

```json5
{
  channels: {
    feishu: {
      appId: "cli_xxx",
      appSecret: "xxx",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "open",
      requireMention: true,
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // Kies dmScope op basis van je isolatiebehoeften:
    // "main" voor automatisch laden van bootstrapbestanden, "per-channel-peer" voor sterkere isolatie
    dmScope: "main",
  },
  bindings: [], // Leeg - dynamische agents worden automatisch gekoppeld
}
```

### Verificatie

Controleer de Gateway-logboeken om te bevestigen dat dynamische aanmaak werkt:

```text
feishu: dynamische agent "feishu-ou_xxxxxx" aanmaken voor gebruiker ou_xxxxxx
  werkruimte: /home/user/.openclaw/workspace-feishu-ou_xxxxxx
  agentmap: /home/user/.openclaw/agents/feishu-ou_xxxxxx/agent
```

Alle aangemaakte werkruimten weergeven:

```bash
ls -la ~/.openclaw/workspace-*
```

### Opmerkingen

- **Werkruimte-isolatie**: Elke gebruiker krijgt een eigen werkruimtemap en agentinstantie. Gebruikers kunnen binnen de normale berichtenstroom elkaars gespreksgeschiedenis of bestanden niet zien.
- **Beveiligingsgrens**: Dit is een isolatiemechanisme voor de berichtencontext, geen beveiligingsgrens tegen vijandige medegebruikers. Het agentproces en de hostomgeving worden gedeeld.
- **Configuratieschrijfbewerkingen moeten ingeschakeld blijven**: Bij het dynamisch aanmaken van agents worden agents en bindingen naar de configuratie geschreven; dit wordt overgeslagen wanneer `channels.feishu.configWrites` `false` is (standaard: ingeschakeld).
- **`bindings` moet leeg zijn**: Dynamische agents registreren automatisch hun eigen bindingen
- **Upgradepad**: Bestaande handmatige bindingen blijven naast dynamische agents werken
- **`session.dmScope` is globaal**: Dit is van invloed op alle kanalen, niet alleen Feishu

## Configuratiereferentie

Volledige configuratie: [Gateway-configuratie](/nl/gateway/configuration)

| Instelling                                               | Beschrijving                                                                         | Standaard                            |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------ |
| `channels.feishu.enabled`                                | Het kanaal in-/uitschakelen                                                          | `true`                               |
| `channels.feishu.domain`                                 | API-domein (`feishu`, `lark` of een `https://`-basis-URL)                              | `feishu`                             |
| `channels.feishu.connectionMode`                         | Gebeurtenistransport (`websocket` of `webhook`)                                      | `websocket`                          |
| `channels.feishu.defaultAccount`                         | Standaardaccount voor uitgaande routering                                            | `default`                            |
| `channels.feishu.verificationToken`                      | Vereist voor Webhook-modus                                                           | -                                    |
| `channels.feishu.encryptKey`                             | Vereist voor Webhook-modus                                                           | -                                    |
| `channels.feishu.webhookPath`                            | Routepad van Webhook                                                                 | `/feishu/events`                     |
| `channels.feishu.webhookHost`                            | Bindingshost van Webhook                                                             | `127.0.0.1`                          |
| `channels.feishu.webhookPort`                            | Bindingspoort van Webhook                                                            | `3000`                               |
| `channels.feishu.accounts.<id>.appId`                    | App-ID                                                                               | -                                    |
| `channels.feishu.accounts.<id>.appSecret`                | App-geheim                                                                           | -                                    |
| `channels.feishu.accounts.<id>.domain`                   | Domeinoverschrijving per account                                                     | `feishu`                             |
| `channels.feishu.accounts.<id>.tts`                      | TTS-overschrijving per account                                                       | `tts`                                |
| `channels.feishu.dmPolicy`                               | DM-beleid (`pairing`, `allowlist`, `open`)                                       | `pairing`                            |
| `channels.feishu.allowFrom`                              | DM-toelatingslijst (lijst met open_id's)                                             | -                                    |
| `channels.feishu.groupPolicy`                            | Groepsbeleid (`open`, `allowlist`, `disabled`)                                  | `allowlist`                          |
| `channels.feishu.groupAllowFrom`                         | Groepstoelatingslijst                                                                | -                                    |
| `channels.feishu.groupSenderAllowFrom`                   | Toelatingslijst voor afzenders die op alle groepen wordt toegepast                   | -                                    |
| `channels.feishu.requireMention`                         | @vermelding in groepen vereisen                                                      | `true` (`false` wanneer beleid `open` is)  |
| `channels.feishu.allowBots`                              | Andere bots accepteren die deze bot vermelden, met bescherming tegen botlussen       | `false`                              |
| `channels.feishu.groups.<chat_id>.requireMention`        | Overschrijving van @vermelding per groep; expliciete ID's laten de groep ook toe in de toelatingslijstmodus | overgenomen                           |
| `channels.feishu.groups.<chat_id>.enabled`               | Een specifieke groep in-/uitschakelen                                                | `true`                               |
| `channels.feishu.groups.<chat_id>.allowFrom`             | Toelatingslijst voor afzenders per groep (overschrijft `groupSenderAllowFrom`)       | -                                    |
| `channels.feishu.groupSessionScope`                      | Toewijzing van groepssessies (`group`, `group_sender`, `group_topic`, `group_topic_sender`) | `group`                              |
| `channels.feishu.replyInThread`                          | Botantwoorden maken onderwerpthreads aan of zetten deze voort (`disabled`, `enabled`) | `disabled`                           |
| `channels.feishu.reactionNotifications`                  | Inkomende reactiegebeurtenissen (`off`, `own`, `all`)                           | `own`                                |
| `channels.feishu.vcAutoJoin`                             | Deelnemen aan VC-vergaderingen waarvoor de bot is uitgenodigd, na normale DM-autorisatie | `false`                              |
| `channels.feishu.dynamicAgentCreation.enabled`           | Automatisch agents per gebruiker aanmaken inschakelen                                | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | Padsjabloon voor dynamische agentwerkruimten                                          | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | Naamsjabloon voor agentmappen                                                        | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | Maximaal aantal aan te maken dynamische agents                                       | onbeperkt                            |
| `channels.feishu.textChunkLimit`                         | Grootte van berichtsegmenten                                                         | `4000`                               |
| `channels.feishu.streaming.chunkMode`                    | Segmentopsplitsing (`length` of `newline`)                                         | `length`                             |
| `channels.feishu.mediaMaxMb`                             | Limiet voor mediagrootte                                                             | `30`                                 |
| `channels.feishu.renderMode`                             | Weergave van antwoorden (`auto`, `raw`, `card`)                                    | `auto`                               |
| `channels.feishu.streaming.mode`                         | Uitvoer van streamingkaarten (`partial` of `off`)                              | `partial`                            |
| `channels.feishu.streaming.block.enabled`                | Antwoordstreaming per voltooid blok                                                  | `false`                              |
| `channels.feishu.typingIndicator`                        | Typreacties verzenden                                                                | `true`                               |
| `channels.feishu.resolveSenderNames`                     | Weergavenamen van afzenders opzoeken                                                 | `true`                               |
| `channels.feishu.configWrites`                           | Door het kanaal geïnitieerde configuratieschrijfbewerkingen toestaan (nodig voor dynamische agents) | `true`                               |
| `channels.feishu.tools.doc`                              | Documenthulpmiddelen inschakelen                                                     | `true`                               |
| `channels.feishu.tools.chat`                             | Hulpmiddelen voor chatinformatie inschakelen                                         | `true`                               |
| `channels.feishu.tools.wiki`                             | Kennisbankhulpmiddelen inschakelen (vereist `doc`)                                  | `true`                               |
| `channels.feishu.tools.drive`                            | Hulpmiddelen voor cloudopslag inschakelen                                            | `true`                               |
| `channels.feishu.tools.perm`                             | Hulpmiddelen voor machtigingsbeheer inschakelen                                      | `false`                              |
| `channels.feishu.tools.scopes`                           | Diagnostisch hulpmiddel voor app-scopes inschakelen                                  | `true`                               |
| `channels.feishu.tools.bitable`                          | Bitable/Base-hulpmiddelen inschakelen                                                | `true`                               |
| `channels.feishu.tools.base`                             | Alias voor `channels.feishu.tools.bitable`; expliciete `bitable` heeft voorrang wanneer beide zijn ingesteld | `true`                               |
| `channels.feishu.accounts.<id>.tools.bitable`            | Poortwachter voor Bitable/Base-hulpmiddelen per account                              | overgenomen                           |
| `channels.feishu.accounts.<id>.tools.base`               | Alias per account voor `tools.bitable`                                               | overgenomen                           |

## Ondersteunde berichttypen

### Ontvangen

- ✅ Tekst
- ✅ Rijke tekst (bericht)
- ✅ Afbeeldingen
- ✅ Bestanden
- ✅ Audio
- ✅ Video/media
- ✅ Stickers

Inkomende Feishu/Lark-audioberichten worden genormaliseerd als mediaplaatsaanduidingen in plaats
van onbewerkte `file_key`-JSON. Wanneer `tools.media.audio` is geconfigureerd, downloadt OpenClaw
de spraaknotitiebron en voert het vóór de agentbeurt gedeelde audiotranscriptie uit,
zodat de agent het gesproken transcript ontvangt. Als Feishu transcriptietekst
rechtstreeks in de audiopayload opneemt, wordt die tekst gebruikt zonder een extra
ASR-aanroep. Zonder provider voor audiotranscriptie ontvangt de agent nog steeds een
`<media:audio>`-plaatsaanduiding plus de opgeslagen bijlage, niet de onbewerkte
Feishu-bronpayload.

### Verzenden

- ✅ Tekst
- ✅ Afbeeldingen
- ✅ Bestanden
- ✅ Audio
- ✅ Video/media
- ✅ Interactieve kaarten (inclusief streamingupdates)
- ⚠️ Rijke tekst (berichtopmaak; ondersteunt niet alle auteursmogelijkheden van Feishu/Lark)

Native Feishu/Lark-audiobubbels gebruiken het Feishu-berichttype `audio` en vereisen
Ogg/Opus-uploadmedia (`file_type: "opus"`). Bestaande `.opus`- en `.ogg`-media
worden rechtstreeks als native audio verzonden. MP3/WAV/M4A en andere waarschijnlijke audioformaten worden
alleen met `ffmpeg` getranscodeerd naar 48kHz Ogg/Opus wanneer het antwoord om
spraakbezorging vraagt (`audioAsVoice` / berichttool `asVoice`, inclusief antwoorden
met TTS-spraaknotities). Gewone MP3-bijlagen blijven reguliere bestanden. Als `ffmpeg` ontbreekt of
de conversie mislukt, valt OpenClaw terug op een bestandsbijlage en wordt de reden gelogd.

### Threads en antwoorden

- ✅ Inline antwoorden
- ✅ Antwoorden in threads
- ✅ Media-antwoorden blijven rekening houden met de thread wanneer op een threadbericht wordt geantwoord

Sessieroutering voor onderwerpgroepen wordt behandeld onder
[Sessiebereik van groepen en onderwerpthreads](#group-session-scope-and-topic-threads).

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) - alle ondersteunde kanalen
- [Koppeling](/nl/channels/pairing) - DM-authenticatie en koppelingsflow
- [Groepen](/nl/channels/groups) - gedrag van groepschats en toegangscontrole via vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) - sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) - toegangsmodel en beveiligingsversterking
