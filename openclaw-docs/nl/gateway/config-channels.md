---
read_when:
    - Een kanaalplugin configureren (authenticatie, toegangsbeheer, meerdere accounts)
    - Problemen met configuratiesleutels per kanaal oplossen
    - DM-beleid, groepsbeleid of vermeldingsfiltering controleren
summary: 'Kanaalconfiguratie: toegangsbeheer, koppeling en sleutels per kanaal voor Slack, Discord, Telegram, WhatsApp, Matrix, iMessage en meer'
title: Configuratie — kanalen
x-i18n:
    generated_at: "2026-07-27T06:14:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e346648287d275d84a9c082a3bb13edaee751d53546d8231dcf1525bf9adafc2
    source_path: gateway/config-channels.md
    workflow: 16
---

Configuratiesleutels per kanaal onder `channels.*`: toegang tot privéberichten en groepen, configuraties met meerdere accounts, vermeldingsfiltering en kanaalspecifieke sleutels voor Slack, Discord, Telegram, WhatsApp, Matrix, iMessage en andere kanaalplugins.

Zie [Configuratiereferentie](/nl/gateway/configuration-reference) voor agents, tools, Gateway-runtime en andere sleutels op het hoogste niveau.

## Kanalen

Elk kanaal start automatisch wanneer de bijbehorende configuratiesectie bestaat (tenzij `enabled: false`). Telegram en iMessage worden meegeleverd in het kernpakket `openclaw`. Andere officiële kanalen (Discord, Slack, WhatsApp, Matrix, Microsoft Teams, IRC, Google Chat, Signal, Mattermost en meer) worden als afzonderlijke plugins geïnstalleerd met `openclaw plugins install <spec>`; zie [Kanalen](/nl/channels) voor de volledige lijst en installatiespecificaties.

### Toegang tot privéberichten en groepen

Alle kanalen ondersteunen beleid voor privéberichten en groepen:

| Beleid voor privéberichten | Gedrag                                                          |
| -------------------------- | --------------------------------------------------------------- |
| `pairing` (standaard) | Onbekende afzenders krijgen een eenmalige koppelingscode; de eigenaar moet deze goedkeuren |
| `allowlist`         | Alleen afzenders in `allowFrom` (of de opslag met gekoppelde toegestane afzenders) |
| `open`         | Alle inkomende privéberichten toestaan (vereist `allowFrom: ["*"]`) |
| `disabled`         | Alle inkomende privéberichten negeren                            |

| Groepsbeleid                | Gedrag                                                  |
| --------------------------- | ------------------------------------------------------- |
| `allowlist` (standaard) | Alleen groepen die overeenkomen met de geconfigureerde toelatingslijst |
| `open`          | Toelatingslijsten voor groepen omzeilen (vermeldingsfiltering blijft van toepassing) |
| `disabled`          | Alle groeps-/kamerberichten blokkeren                   |

<Note>
`channels.defaults.groupPolicy` stelt de standaardwaarde in wanneer `groupPolicy` van een provider niet is ingesteld.
Koppelingscodes verlopen na 1 uur. Het aantal openstaande koppelingsverzoeken is beperkt tot **3 per account** (afgebakend per kanaal en account-id).
Als een providerblok volledig ontbreekt (`channels.<provider>` ontbreekt), valt het groepsbeleid tijdens runtime terug op `allowlist` (standaard blokkeren) met een waarschuwing bij het opstarten.
</Note>

### Modeloverschrijvingen per kanaal

Gebruik `channels.modelByChannel` om specifieke kanaal-id's of peers voor privéberichten aan een model te koppelen. Waarden accepteren `provider/model` of geconfigureerde modelaliassen. De kanaaltoewijzing is alleen van toepassing wanneer een sessie nog geen actieve modeloverschrijving heeft (bijvoorbeeld een die via `/model` is ingesteld).

Voor groeps-/threadgesprekken zijn de sleutels kanaalspecifieke groeps-id's, onderwerp-id's of kanaalnamen. Voor gesprekken via privéberichten zijn de sleutels peer-id's die zijn afgeleid van de afzenderidentiteit van het kanaal (`nativeDirectUserId`, `origin.from`, `origin.to`, `OriginatingTo`, `From` of `SenderId`). De exacte sleutelvorm hangt af van het kanaal:

| Kanaal   | Sleutelvorm voor privéberichten | Voorbeeld                                    |
| -------- | -------------------------------- | -------------------------------------------- |
| Discord  | onbewerkte gebruikers-id         | `987654321`                           |
| Feishu   | `feishu:ou_...`                | `feishu:ou_a8b6cab7e945387de5f253775d9b4d85`                           |
| Matrix   | Matrix-gebruikers-id              | `@user:matrix.org`                           |
| Slack    | `user:U...`                | `user:U12345`                           |
| Telegram | onbewerkte gebruikers-id         | `123456789`                           |
| WhatsApp | telefoonnummer of JID             | `15551234567`                           |

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-5.6-sol",
        "user:U12345": "openai/gpt-5.4-mini",
      },
      telegram: {
        "-1001234567890": "openai/gpt-5.4-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
        "123456789": "openai/gpt-4.1",
      },
    },
  },
}
```

Sleutels die specifiek zijn voor privéberichten komen alleen overeen in gesprekken via privéberichten; ze hebben geen invloed op routering van groepen/threads.

### Kanaalstandaarden en Heartbeat

Gebruik `channels.defaults` voor gedeeld groepsbeleid, impliciete vermeldingen en Heartbeat-gedrag voor alle providers:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      implicitMentions: {
        replyToBot: true,
        quotedBot: true,
        threadParticipation: true,
      },
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: terugvalbeleid voor groepen wanneer `groupPolicy` op providerniveau niet is ingesteld.
- `channels.defaults.contextVisibility`: standaardmodus voor de zichtbaarheid van aanvullende context voor alle kanalen. Waarden: `all` (standaard, alle aangehaalde/thread-/geschiedeniscontext opnemen), `allowlist` (alleen context van afzenders op de toelatingslijst opnemen), `allowlist_quote` (hetzelfde als de toelatingslijst, maar expliciete citaat-/antwoordcontext behouden). Overschrijving per kanaal: `channels.<channel>.contextVisibility`.
- `channels.defaults.implicitMentions`: bepaalt welke ondersteunde inkomende feiten als vermeldingen gelden. `replyToBot`, `quotedBot` en `threadParticipation` hebben elk standaard de waarde `true`, waardoor het huidige gedrag behouden blijft. Overschrijf dit per kanaal met `channels.<channel>.implicitMentions` of per account met `channels.<channel>.accounts.<id>.implicitMentions`; elke vlag wordt onafhankelijk opgelost in de volgorde account -> kanaal -> standaardwaarden. De namen zijn positief geformuleerd: stel een vlag in op `false` om te voorkomen dat dit feit de vermeldingsfiltering omzeilt. Expliciete systeemeigen vermeldingen zijn altijd toegestaan en een vlag heeft geen effect wanneer het kanaal dat feit niet produceert. Zie [Vermeldingsfiltering](/nl/channels/groups#mention-gating-default) voor de huidige producentenmatrix. Deze instellingen wijzigen de uitgaande antwoord-/threadmodi of de afhandeling van geautoriseerde opdrachten niet.
- `channels.defaults.heartbeat.showOk`: gezonde kanaalstatussen opnemen in de Heartbeat-uitvoer (standaard `false`).
- `channels.defaults.heartbeat.showAlerts`: verslechterde/foutstatussen opnemen in de Heartbeat-uitvoer (standaard `true`).
- `channels.defaults.heartbeat.useIndicator`: compacte Heartbeat-uitvoer in indicatorstijl weergeven (standaard `true`).

### WhatsApp

WhatsApp werkt via het webkanaal van de Gateway (Baileys Web). Het start automatisch wanneer er een gekoppelde sessie bestaat.

```json5
{
  web: {
    enabled: true,
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" }, // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // blue ticks (false in self-chat mode)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

- Vermeldingen van `bindings[]` op het hoogste niveau met `type: "acp"` configureren permanente ACP-koppelingen voor WhatsApp-privéberichten en -groepen. Gebruik een rechtstreeks E.164-nummer of een WhatsApp-groeps-JID in `match.peer.id`. De veldsemantiek wordt gedeeld in [ACP-agents](/nl/tools/acp-agents#persistent-channel-bindings).

<Accordion title="WhatsApp met meerdere accounts">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- Uitgaande opdrachten gebruiken standaard account `default` als dat aanwezig is; anders de eerste geconfigureerde account-id (gesorteerd).
- De optionele instelling `channels.whatsapp.defaultAccount` overschrijft die terugvalselectie van het standaardaccount wanneer deze overeenkomt met een geconfigureerde account-id.
- De verouderde Baileys-authenticatiemap voor één account wordt door `openclaw doctor` gemigreerd naar `whatsapp/default`.
- Overschrijvingen per account: `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: { mode: "partial" }, // off | partial | block | progress (default: partial)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      apiRoot: "https://api.telegram.org",
      trustedLocalFileRoots: ["/srv/telegram-bot-api-data"],
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Bottoken: `channels.telegram.botToken` of `channels.telegram.tokenFile` (alleen een normaal bestand; symbolische koppelingen worden geweigerd), met `TELEGRAM_BOT_TOKEN` als terugvaloptie voor het standaardaccount.
- `apiRoot` is uitsluitend de hoofd-URL van de Telegram Bot API. Gebruik `https://api.telegram.org` of je zelfgehoste/proxyhoofd-URL, niet `https://api.telegram.org/bot<TOKEN>`; `openclaw doctor --fix` verwijdert een onbedoeld achtervoegsel `/bot<TOKEN>` aan het einde.
- Voor een zelfgehoste Bot API-server in de modus `--local` vermeldt `trustedLocalFileRoots` de hostpaden die OpenClaw mag lezen. Koppel het gegevensvolume van de server aan de OpenClaw-host en configureer de hoofdmap met gegevens of de map per token; containerpaden onder `/var/lib/telegram-bot-api` worden aan die hoofdmappen toegewezen. Andere absolute paden blijven geweigerd.
- De optionele instelling `channels.telegram.defaultAccount` overschrijft de selectie van het standaardaccount wanneer deze overeenkomt met een geconfigureerde account-id.
- Stel in configuraties met meerdere accounts (2+ account-id's) een expliciete standaardwaarde in (`channels.telegram.defaultAccount` of `channels.telegram.accounts.default`) om terugvalroutering te vermijden; `openclaw doctor` waarschuwt wanneer deze ontbreekt of ongeldig is.
- `configWrites: false` blokkeert door Telegram geïnitieerde configuratieschrijfacties (migraties van supergroep-id's, `/config set|unset`).
- Vermeldingen van `bindings[]` op het hoogste niveau met `type: "acp"` configureren permanente ACP-koppelingen voor forumonderwerpen (gebruik de canonieke `chatId:topic:topicId` in `match.peer.id`). De veldsemantiek wordt gedeeld in [ACP-agents](/nl/tools/acp-agents#persistent-channel-bindings).
- Telegram-streamvoorbeelden gebruiken `sendMessage` + `editMessageText` (werkt in privé- en groepschats).
- `network.dnsResultOrder` heeft standaard de waarde `"ipv4first"` om veelvoorkomende IPv6-ophaalfouten te voorkomen.
- Beleid voor nieuwe pogingen: zie [Beleid voor nieuwe pogingen](/nl/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Alleen korte antwoorden.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      suppressEmbeds: true,
      streaming: {
        mode: "progress", // off | partial | block | progress (standaard voor Discord: progress)
        chunkMode: "length", // length | newline
        progress: {
          label: "auto",
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: true,
        },
      },
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSessions: true,
        defaultSpawnContext: "fork",
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- Token: `channels.discord.token`, met `DISCORD_BOT_TOKEN` als terugvaloptie voor het standaardaccount.
- Directe uitgaande aanroepen die een expliciete Discord-`token` opgeven, gebruiken die token voor de aanroep; instellingen voor nieuwe pogingen en beleid van het account zijn nog steeds afkomstig van het geselecteerde account in de actieve runtimesnapshot.
- Optioneel overschrijft `channels.discord.defaultAccount` de selectie van het standaardaccount wanneer deze overeenkomt met een geconfigureerde account-id.
- Gebruik `user:<id>` (DM) of `channel:<id>` (guild-kanaal) voor afleveringsdoelen; losse numerieke id's worden geweigerd.
- Guild-slugs bestaan uit kleine letters, waarbij spaties zijn vervangen door `-`; kanaalsleutels gebruiken de naam als slug (zonder `#`). Geef de voorkeur aan guild-id's.
- Berichten die door bots zijn opgesteld, worden standaard genegeerd. `allowBots: true` schakelt ze in; gebruik `allowBots: "mentions"` om alleen botberichten te accepteren waarin de bot wordt vermeld (eigen berichten worden nog steeds gefilterd).
- Kanalen die inkomende, door bots opgestelde berichten ondersteunen, kunnen gedeelde [bescherming tegen botlussen](/nl/channels/bot-loop-protection) gebruiken. Stel `channels.defaults.botLoopProtection` in voor basisbudgetten per paar en overschrijf daarna alleen het kanaal of account wanneer één oppervlak andere limieten nodig heeft.
- `channels.discord.guilds.<id>.ignoreOtherMentions` (en kanaaloverschrijvingen) verwijdert berichten die een andere gebruiker of rol vermelden, maar niet de bot (met uitzondering van @everyone/@here).
- `channels.discord.mentionAliases` koppelt vóór verzending stabiele uitgaande `@handle`-tekst aan Discord-gebruikers-id's, zodat bekende teamgenoten deterministisch kunnen worden vermeld, zelfs wanneer de tijdelijke directorycache leeg is. Overschrijvingen per account staan onder `channels.discord.accounts.<accountId>.mentionAliases`.
- `maxLinesPerMessage` (standaard `17`) splitst lange berichten, zelfs wanneer ze minder dan 2000 tekens bevatten.
- `channels.discord.suppressEmbeds` is standaard `true`, zodat uitgaande URL's niet worden uitgevouwen tot Discord-linkvoorbeelden, tenzij dit wordt uitgeschakeld. Expliciete `embeds`-payloads worden nog steeds normaal verzonden; toolaanroepen per bericht kunnen dit overschrijven met `suppressEmbeds`.
- `channels.discord.threadBindings` regelt routering die aan Discord-threads is gekoppeld:
  - `enabled`: Discord-overschrijving voor functies van aan threads gekoppelde sessies (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` en gekoppelde aflevering/routering)
  - `idleHours`: Discord-overschrijving voor automatisch verlies van focus na inactiviteit, in uren (`0` schakelt dit uit)
  - `maxAgeHours`: Discord-overschrijving voor de harde maximale leeftijd in uren (`0` schakelt dit uit)
  - `spawnSessions`: schakelaar voor `sessions_spawn({ thread: true })` en het automatisch maken/koppelen van threads bij het starten van ACP-threads (standaard: `true`)
  - `defaultSpawnContext`: systeemeigen subagentcontext voor aan threads gekoppelde starts (standaard `"fork"`)
- Items op het hoogste niveau in `bindings[]` met `type: "acp"` configureren persistente ACP-koppelingen voor kanalen en threads (gebruik de kanaal-/thread-id in `match.peer.id`). De semantiek van de velden wordt gedeeld in [ACP-agenten](/nl/tools/acp-agents#persistent-channel-bindings).
- `channels.discord.ui.components.accentColor` stelt de accentkleur in voor containers met Discord-componenten v2.
- `channels.discord.agentComponents.ttlMs` bepaalt hoelang callbacks van verzonden Discord-componenten geregistreerd blijven. Standaard `1800000` (30 minuten), maximaal `86400000` (24 uur). Overschrijvingen per account staan onder `channels.discord.accounts.<accountId>.agentComponents.ttlMs`. Geef de voorkeur aan de kortste TTL die bij de workflow past.
- `channels.discord.voice` schakelt gesprekken in Discord-spraakkanalen en optioneel automatisch deelnemen + LLM- + TTS-overschrijvingen in. Discord-configuraties met alleen tekst laten spraak standaard uitgeschakeld; stel `channels.discord.voice.enabled=true` in om dit in te schakelen.
- `channels.discord.voice.model` overschrijft optioneel het LLM-model dat wordt gebruikt voor antwoorden in Discord-spraakkanalen.
- `channels.discord.voice.daveEncryption` (standaard `true`) en `channels.discord.voice.decryptionFailureTolerance` (standaard `24`) worden doorgegeven aan de DAVE-opties van `@discordjs/voice`.
- `channels.discord.voice.connectTimeoutMs` regelt de initiële wachttijd op `@discordjs/voice` Ready voor `/vc join` en pogingen om automatisch deel te nemen (standaard `30000`).
- `channels.discord.voice.reconnectGraceMs` bepaalt hoelang een verbroken spraaksessie erover mag doen om signalering voor opnieuw verbinden te starten voordat OpenClaw de sessie vernietigt (standaard `15000`).
- Het afspelen van Discord-spraak wordt niet onderbroken door een gebeurtenis waarbij een andere gebruiker begint te spreken. Om feedbacklussen te voorkomen, negeert OpenClaw nieuwe spraakopname terwijl TTS wordt afgespeeld.
- OpenClaw probeert daarnaast de ontvangst van spraak te herstellen door een spraaksessie te verlaten en opnieuw deel te nemen na herhaalde ontsleutelingsfouten.
- `channels.discord.streaming` is de canonieke sleutel voor de streammodus. Discord gebruikt standaard `streaming.mode: "progress"`, zodat de voortgang van tools/werk in één bewerkt voorbeeldbericht verschijnt; stel `streaming.mode: "off"` in om dit uit te schakelen. Verouderde platte sleutels (`streamMode`, `chunkMode`, `blockStreaming`, `draftChunk`, `blockStreamingCoalesce`) worden tijdens runtime niet meer gelezen; voer `openclaw doctor --fix` uit om persistente configuratie te migreren.
- `channels.discord.autoPresence` koppelt runtimebeschikbaarheid aan de aanwezigheid van de bot (gezond => online, verminderd => idle, uitgeput => dnd) en staat optionele overschrijvingen van statustekst toe.
- `channels.discord.guilds.<id>.presenceEvents` routeert aankomsten van beschikbare personen als agentsysteemgebeurtenissen naar één geconfigureerd Discord-kanaal. In aanmerking komende leden moeten `channelId` kunnen bekijken; openbare threads nemen de zichtbaarheid van hun bovenliggende kanaal over, terwijl voor privéthreads daarnaast lidmaatschap of Manage Threads vereist is. `users` kan die doelgroep verder beperken. Dit initialiseert de momenteel online leden vanuit volledige `GUILD_CREATE`-snapshots, routeert waargenomen overgangen van offline naar online en behandelt een eerste later onlinesignaal voor een nog niet waargenomen lid als nieuw beschikbaar, zonder te stellen of die persoon online kwam of pas na de snapshot deelnam. Guilds boven Discords snapshotlimiet van 75,000 leden vereisen eerst een expliciete offline-update. Instellingen voor beperking: `reconnectSuppressSeconds` (stilteperiode na een nieuwe Gateway-sessie terwijl de aanwezigheidsstatus van de guild opnieuw wordt opgebouwd, standaard 300, `0` schakelt dit uit) en `burstLimit`/`burstWindowSeconds` (limiet per guild voor de frequentie van succesvol in de wachtrij geplaatste gebeurtenissen, standaard 8 gebeurtenissen per voortschrijdend venster van 60s). Hervatte sessies starten de onderdrukkingsperiode voor opnieuw verbinden niet. De bestaande afkoelperiode van acht uur voor hernieuwde begroeting per gebruiker blijft gelden. Dit vereist `channels.discord.intents.presence=true`, de bevoorrechte Presence Intent in Discords Developer Portal en een ingeschakelde Heartbeat voor de agent.
- `channels.discord.dangerouslyAllowNameMatching` schakelt overeenkomsten op basis van veranderlijke namen/tags opnieuw in (compatibiliteitsmodus voor noodgevallen).
- `channels.discord.execApprovals`: systeemeigen Discord-aflevering van uitvoeringsgoedkeuringen en autorisatie van goedkeurders.
  - `enabled`: `true`, `false` of `"auto"` (standaard). In de automatische modus worden uitvoeringsgoedkeuringen geactiveerd wanneer goedkeurders kunnen worden herleid uit `approvers` of `commands.ownerAllowFrom`.
  - `approvers`: Discord-gebruikers-id's die uitvoeringsverzoeken mogen goedkeuren. Valt terug op `commands.ownerAllowFrom` wanneer dit wordt weggelaten.
  - `agentFilter`: optionele acceptatielijst voor agent-id's. Laat dit weg om goedkeuringen voor alle agenten door te sturen.
  - `sessionFilter`: optionele patronen voor sessiesleutels (subtekenreeks of reguliere expressie).
  - `target`: waar goedkeuringsverzoeken naartoe worden verzonden. `"dm"` (standaard) verzendt ze naar DM's van goedkeurders, `"channel"` verzendt ze naar het oorspronkelijke kanaal en `"both"` verzendt ze naar beide. Wanneer het doel `"channel"` bevat, kunnen de knoppen alleen worden gebruikt door herleide goedkeurders.
  - `cleanupAfterResolve`: wanneer `true`, worden goedkeurings-DM's verwijderd na goedkeuring, afwijzing of time-out.

**Modi voor reactiemeldingen:** `off` (geen), `own` (berichten van de bot, standaard), `all` (alle berichten), `allowlist` (van `guilds.<id>.users` op alle berichten).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- JSON van serviceaccount: inline (`serviceAccount`) of op basis van een bestand (`serviceAccountFile`).
- `serviceAccount` accepteert rechtstreeks een SecretRef.
- Terugvalopties via omgevingsvariabelen: `GOOGLE_CHAT_SERVICE_ACCOUNT` of `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (alleen het standaardaccount).
- Gebruik `spaces/<spaceId>` of `users/<userId>` voor afleveringsdoelen.
- `channels.googlechat.dangerouslyAllowNameMatching` schakelt overeenkomsten op basis van veranderlijke e-mailprincipals opnieuw in (compatibiliteitsmodus voor noodgevallen).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      socketMode: {
        clientPingTimeout: 15000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { enabled: true, requireMention: true, allowBots: false },
        "#general": {
          enabled: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Alleen korte antwoorden.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // uit | eerste | alle | gebundeld
      thread: {
        historyScope: "thread", // thread | kanaal
        inheritParent: false,
        initialHistoryLimit: 20,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      unfurlLinks: false,
      unfurlMedia: false,
      textChunkLimit: 4000,
      streaming: {
        mode: "partial", // uit | gedeeltelijk | blok | voortgang
        chunkMode: "length", // lengte | nieuwe regel
        nativeTransport: true, // gebruik de native streaming-API van Slack wanneer mode=partial
      },
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | kanaal | beide
      },
    },
  },
}
```

- **Socket-modus** vereist zowel `botToken` als `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` voor de terugval naar omgevingsvariabelen van het standaardaccount).
- **HTTP-modus** vereist `botToken` plus `signingSecret` (op rootniveau of per account).
- **Gebruikersidentiteit** (`identity: "user"`) plaatst en leest berichten als de autoriserende persoon. Hiervoor zijn `userToken` plus `appToken` in Socket-modus vereist, of `userToken` plus `signingSecret` in HTTP-modus. Er is geen bottoken of botgebruiker vereist. Zie [Gebruikersidentiteit](/nl/channels/slack#user-identity-post-as-a-real-person) voor gebruikersbereiken en gebeurtenisabonnementen.
- `enterpriseOrgInstall: true` meldt een account aan voor het organisatiebrede gebeurtenispad van Slack Enterprise Grid. Bij het opstarten wordt het bottoken geverifieerd met `auth.test` en
  mislukt het proces wanneer de geconfigureerde modus niet overeenkomt met de installatie-identiteit van Slack.
  Enterprise-DM's moeten zijn uitgeschakeld of `dmPolicy: "open"` gebruiken met een geldige
  `allowFrom: ["*"]`. Kanaal- en gebruikersbeleid moeten stabiele Slack-ID's gebruiken;
  veranderlijke namen en niet-ondersteunde kanaalvoorvoegsels laten het opstarten mislukken. V1 verwerkt alleen
  rechtstreekse Socket-modus- of HTTP-gebeurtenissen `message` en `app_mention` met onmiddellijke
  antwoorden; relay, opdrachten, interacties, App Home, listeners voor reactiegebeurtenissen,
  pins, actietools, native goedkeuringen, bindings, uitgestelde bezorging en
  proactieve verzendingen zijn niet beschikbaar. Door de listener beheerde bevestigingen, typindicatoren en
  statusreacties blijven beschikbaar met `reactions:write`; meldingen voor inkomende reacties
  en reactie-actietools zijn niet beschikbaar. Zie
  [Organisatiebrede installaties van Enterprise Grid](/nl/channels/slack#enterprise-grid-org-wide-installs)
  voor het manifest met minimale bevoegdheden, de instelworkflow en alle beperkingen.
- `socketMode` geeft afstemmingsinstellingen voor het Socket-modus-transport van de Slack SDK door aan de openbare Bolt-receiver-API. Gebruik dit alleen bij onderzoek naar time-outs voor ping/pong of verouderd websocketgedrag. `clientPingTimeout` is standaard `15000`; `serverPingTimeout` en `pingPongLoggingEnabled` worden alleen doorgegeven wanneer ze zijn geconfigureerd.
- `botToken`, `appToken`, `signingSecret` en `userToken` accepteren plattetekstreekswaarden
  of SecretRef-objecten.
- Momentopnamen van Slack-accounts tonen per referentie bron-/statusvelden zoals
  `botTokenSource`, `botTokenStatus`, `userTokenSource`, `userTokenStatus`,
  `appTokenStatus` en, in HTTP-modus, `signingSecretStatus`.
  `configured_unavailable` betekent dat het account
  via SecretRef is geconfigureerd, maar dat het huidige opdracht-/runtimepad
  de geheime waarde niet kon oplossen.
- `configWrites: false` blokkeert door Slack geïnitieerde configuratieschrijfbewerkingen.
- De optionele `channels.slack.defaultAccount` overschrijft de selectie van het standaardaccount wanneer deze overeenkomt met een geconfigureerd account-ID.
- `channels.slack.streaming.mode` is de canonieke sleutel voor de Slack-streammodus (standaard `"partial"`). `channels.slack.streaming.nativeTransport` beheert het native streamingtransport van Slack (standaard `true`). Verouderde waarden voor `streamMode`, de booleaanse `streaming`, `chunkMode`, `blockStreaming`, `blockStreamingCoalesce` en `nativeStreaming` worden tijdens runtime niet meer gelezen; voer `openclaw doctor --fix` uit om opgeslagen configuratie naar `streaming.{mode,chunkMode,block.enabled,block.coalesce,nativeTransport}` te migreren.
- `unfurlLinks` en `unfurlMedia` geven de booleaanse instellingen van Slack voor het uitvouwen van `chat.postMessage`-links en media door voor botantwoorden. `unfurlLinks` is standaard `false`, zodat uitgaande botlinks niet inline worden uitgevouwen tenzij dit is ingeschakeld; `unfurlMedia` wordt weggelaten tenzij het is geconfigureerd. Stel een van beide waarden in bij `channels.slack.accounts.<accountId>` om de waarde op het hoogste niveau voor één account te overschrijven.
- Gebruik `user:<id>` (DM) of `channel:<id>` voor bezorgingsdoelen.

**Modi voor reactiemeldingen:** `off`, `own` (standaard), `all`, `allowlist` (van `reactionAllowlist`).

**Isolatie van threadsessies:** `thread.historyScope` is per thread (standaard) of wordt gedeeld binnen het kanaal. `thread.inheritParent` kopieert het transcript van het bovenliggende kanaal naar nieuwe threads. `thread.initialHistoryLimit` (standaard `20`) beperkt hoeveel bestaande threadberichten worden opgehaald wanneer een nieuwe threadsessie begint; `0` schakelt het ophalen van threadgeschiedenis uit.

- Voor native streaming van Slack en de Slack-assistentstatus "is typing..." in threads is een doelthread voor het antwoord vereist. DM's op het hoogste niveau blijven standaard buiten threads, zodat ze nog steeds kunnen streamen via Slack-conceptvoorbeelden die worden geplaatst en bewerkt, in plaats van het native stream-/statusvoorbeeld in threadstijl weer te geven.
- `typingReaction` voegt tijdelijk een reactie toe aan het inkomende Slack-bericht terwijl een antwoord wordt uitgevoerd en verwijdert deze na voltooiing. Gebruik een Slack-emoji-shortcode zoals `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: Slack-native bezorging aan de goedkeuringsclient en autorisatie van uitvoeringsgoedkeurders. Hetzelfde schema als Discord: `enabled` (`true`/`false`/`"auto"`), `approvers` (Slack-gebruikers-ID's), `agentFilter`, `sessionFilter` en `target` (`"dm"`, `"channel"` of `"both"`). Plugin-goedkeuringen kunnen dit native clientpad gebruiken voor verzoeken die vanuit Slack afkomstig zijn wanneer Slack Plugin-goedkeurders kunnen worden bepaald; Slack-native bezorging van Plugin-goedkeuringen kan ook worden ingeschakeld via `approvals.plugin` voor sessies die vanuit Slack afkomstig zijn of voor Slack-doelen. Plugin-goedkeuringen gebruiken Slack Plugin-goedkeurders uit `allowFrom` en standaardroutering, niet uitvoeringsgoedkeurders.

| Actiegroep  | Standaard   | Opmerkingen                         |
| ------------ | ----------- | ----------------------------------- |
| reactions    | ingeschakeld | Reageren + reacties weergeven       |
| messages     | ingeschakeld | Lezen/verzenden/bewerken/verwijderen |
| pins         | ingeschakeld | Vastzetten/losmaken/weergeven       |
| memberInfo   | ingeschakeld | Lidgegevens                         |
| emojiList    | ingeschakeld | Lijst met aangepaste emoji's        |

### Mattermost

Mattermost wordt als afzonderlijke Plugin geïnstalleerd, op dezelfde manier als Discord, Slack en WhatsApp:

```bash
openclaw plugins install @openclaw/mattermost
```

Controleer [npmjs.com/package/@openclaw/mattermost](https://www.npmjs.com/package/@openclaw/mattermost) voor de huidige dist-tags voordat je een versie vastzet.

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // bij aanroep | bij bericht | bij teken
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // expliciet inschakelen
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Optionele expliciete URL voor implementaties met een reverse proxy/openbare toegang
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" },
    },
  },
}
```

Chatmodi: `oncall` (antwoorden bij een @-vermelding, standaard), `onmessage` (elk bericht), `onchar` (berichten die met het activeringsvoorvoegsel beginnen).

Wanneer native opdrachten van Mattermost zijn ingeschakeld:

- `commands.callbackPath` moet een pad zijn (bijvoorbeeld `/api/channels/mattermost/command`), geen volledige URL.
- `commands.callbackUrl` moet worden omgezet naar het Gateway-eindpunt van OpenClaw en bereikbaar zijn vanaf de Mattermost-server.
- Native slash-callbacks worden geverifieerd met de tokens per opdracht die
  Mattermost tijdens de registratie van slash-opdrachten retourneert. Als de registratie mislukt of er geen
  opdrachten zijn geactiveerd, wijst OpenClaw callbacks af met
  `Unauthorized: invalid command token.`
- Voor privé-, tailnet- of interne callbackhosts kan Mattermost vereisen
  dat `ServiceSettings.AllowedUntrustedInternalConnections` de callbackhost of het callbackdomein bevat.
  Gebruik host-/domeinwaarden, geen volledige URL's.
- `channels.mattermost.configWrites`: door Mattermost geïnitieerde configuratieschrijfbewerkingen toestaan of weigeren.
- `channels.mattermost.requireMention`: `@mention` vereisen voordat in kanalen wordt geantwoord.
- `channels.mattermost.groups.<channelId>.requireMention`: overschrijving per kanaal van de vermeldingsvereiste (`"*"` voor standaard).
- De optionele `channels.mattermost.defaultAccount` overschrijft de selectie van het standaardaccount wanneer deze overeenkomt met een geconfigureerd account-ID.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // optionele accountbinding
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // uit | eigen | alle | toelatingslijst
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**Modi voor reactiemeldingen:** `off`, `own` (standaard), `all`, `allowlist` (van `reactionAllowlist`).

- `channels.signal.account`: het opstarten van het kanaal vastzetten op de identiteit van een specifiek Signal-account.
- `channels.signal.configWrites`: door Signal geïnitieerde configuratieschrijfbewerkingen toestaan of weigeren.
- De optionele `channels.signal.defaultAccount` overschrijft de selectie van het standaardaccount wanneer deze overeenkomt met een geconfigureerd account-ID.

### iMessage

OpenClaw start `imsg rpc` (JSON-RPC via stdio). Er is geen daemon of poort vereist. Dit is het voorkeurspad voor nieuwe OpenClaw iMessage-configuraties wanneer de host toegang tot de Berichten-database en automatiseringsmachtigingen kan verlenen.

Ondersteuning voor BlueBubbles is verwijderd. `channels.bluebubbles` is geen ondersteund runtimeconfiguratieoppervlak in de huidige OpenClaw-versie. Migreer oude configuraties naar `channels.imessage`; gebruik [Verwijdering van BlueBubbles en het imsg-pad voor iMessage](/nl/announcements/bluebubbles-imessage) voor de korte versie en [Overstappen vanaf BlueBubbles](/nl/channels/imessage-from-bluebubbles) voor de volledige vertaaltabel.

Als de Gateway niet wordt uitgevoerd op de Mac waarop bij Berichten is ingelogd, behoud je `channels.imessage.enabled=true` en stel je `channels.imessage.cliPath` in op een SSH-wrapper die `imsg "$@"` op die Mac uitvoert. Het lokale standaardpad `imsg` is alleen voor macOS.

Voordat je voor productieverzendingen op een SSH-wrapper vertrouwt, moet je een uitgaande `imsg send` via precies die wrapper verifiëren. In sommige macOS TCC-statussen wordt Messages Automation toegewezen aan `/usr/libexec/sshd-keygen-wrapper`, waardoor leesbewerkingen en controles kunnen werken terwijl verzendingen mislukken met AppleEvents `-1743`; zie het gedeelte over probleemoplossing voor de SSH-wrapper op [iMessage](/nl/channels/imessage).

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      sendTransport: "auto",
      region: "US",
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
      },
    },
  },
}
```

- Optionele `channels.imessage.defaultAccount` overschrijft de standaardaccountselectie wanneer deze overeenkomt met een geconfigureerde account-id.
- Vereist volledige schijftoegang tot de Messages-database.
- Geef de voorkeur aan `chat_id:<id>`-doelen. Gebruik `imsg chats --limit 20` om chats weer te geven.
- `cliPath` kan naar een SSH-wrapper verwijzen; stel `remoteHost` (`host` of `user@host`) in om bijlagen via SCP op te halen.
- `attachmentRoots` en `remoteAttachmentRoots` beperken paden voor inkomende bijlagen (standaard: `/Users/*/Library/Messages/Attachments`).
- SCP gebruikt strikte controle van hostsleutels; zorg er daarom voor dat de hostsleutel van de relayhost al in `~/.ssh/known_hosts` staat.
- `channels.imessage.configWrites`: schrijfwijzigingen aan de configuratie die vanuit iMessage worden gestart, toestaan of weigeren.
- `channels.imessage.sendTransport`: gewenst `imsg`-RPC-verzendtransport voor normale uitgaande antwoorden. `auto` (standaard) gebruikt de IMCore-bridge voor bestaande chats wanneer deze actief is en valt daarna terug op AppleScript; `bridge` vereist bezorging via de privé-API; `applescript` dwingt het openbare automatiseringspad van Messages af.
- `channels.imessage.actions.*`: privé-API-acties inschakelen die ook door `imsg status` / `openclaw channels status --probe` worden begrensd.
- `channels.imessage.includeAttachments` is standaard uitgeschakeld; stel dit in op `true` voordat je inkomende media in agentbeurten verwacht.
- Inkomend herstel na een herstart van de bridge/Gateway verloopt automatisch (GUID-deduplicatie plus een leeftijdsgrens voor verouderde achterstanden). Bestaande `channels.imessage.catchup.enabled: true`-configuraties worden nog steeds gerespecteerd als een verouderd compatibiliteitsprofiel; `catchup` is standaard uitgeschakeld.
- `channels.imessage.groups`: groepsregister en instellingen per groep. Configureer met `groupPolicy: "allowlist"` expliciete `chat_id`-sleutels of een jokertekenvermelding `"*"`, zodat groepsberichten de registerpoort kunnen passeren.
- Vermeldingen op het hoogste niveau in `bindings[]` met `type: "acp"` kunnen iMessage-gesprekken aan permanente ACP-sessies koppelen. Gebruik een genormaliseerde handle of een expliciet chatdoel (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) in `match.peer.id`. Semantiek van gedeelde velden: [ACP-agents](/nl/tools/acp-agents#persistent-channel-bindings).

<Accordion title="Voorbeeld van een iMessage SSH-wrapper">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix wordt door een Plugin ondersteund en geconfigureerd onder `channels.matrix`.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- Tokenauthenticatie gebruikt `accessToken`; wachtwoordauthenticatie gebruikt `userId` + `password`.
- `channels.matrix.proxy` leidt Matrix-HTTP-verkeer via een expliciete HTTP(S)-proxy. Benoemde accounts kunnen dit overschrijven met `channels.matrix.accounts.<id>.proxy`.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` staat privé-/interne homeservers toe. `proxy` en deze netwerkopt-in zijn onafhankelijke instellingen.
- `channels.matrix.defaultAccount` selecteert het voorkeursaccount in configuraties met meerdere accounts.
- `channels.matrix.autoJoin` is standaard `"off"`, waardoor uitnodigingen voor ruimten en nieuwe DM-achtige uitnodigingen worden genegeerd totdat je `autoJoin: "allowlist"` instelt met `autoJoinAllowlist` of `autoJoin: "always"`.
- `channels.matrix.execApprovals`: Matrix-eigen levering van uitvoeringsgoedkeuringen en autorisatie van goedkeurders.
  - `enabled`: `true`, `false` of `"auto"` (standaard). In de automatische modus worden uitvoeringsgoedkeuringen geactiveerd wanneer goedkeurders via `approvers` of `commands.ownerAllowFrom` kunnen worden bepaald.
  - `approvers`: Matrix-gebruikers-id's (bijvoorbeeld `@owner:example.org`) die uitvoeringsverzoeken mogen goedkeuren.
  - `agentFilter`: optionele toestaanlijst met agent-id's. Laat dit weg om goedkeuringen voor alle agents door te sturen.
  - `sessionFilter`: optionele patronen voor sessiesleutels (subtekenreeks of reguliere expressie).
  - `target`: waar goedkeuringsprompts naartoe worden gestuurd. `"dm"` (standaard), `"channel"` (de ruimte van herkomst) of `"both"`.
  - Overschrijvingen per account: `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` bepaalt hoe Matrix-DM's in sessies worden gegroepeerd: `per-user` (standaard) deelt per gerouteerde peer, terwijl `per-room` elke DM-ruimte isoleert.
- Matrix-statuscontroles en live-directoryzoekopdrachten gebruiken hetzelfde proxybeleid als runtimeverkeer.
- De volledige Matrix-configuratie, doelregels en installatievoorbeelden zijn gedocumenteerd in [Matrix](/nl/channels/matrix).

### Microsoft Teams

Microsoft Teams wordt door een Plugin ondersteund en geconfigureerd onder `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // zie /channels/msteams
    },
  },
}
```

- Belangrijkste sleutelpaden die hier worden behandeld: `channels.msteams`, `channels.msteams.configWrites`.
- De volledige Teams-configuratie (inloggegevens, Webhook, DM-/groepsbeleid, overschrijvingen per team/per kanaal) is gedocumenteerd in [Microsoft Teams](/nl/channels/msteams).

### IRC

IRC wordt door een Plugin ondersteund en geconfigureerd onder `channels.irc`.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- Belangrijkste sleutelpaden die hier worden behandeld: `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- Optionele `channels.irc.defaultAccount` overschrijft de standaardaccountselectie wanneer deze overeenkomt met een geconfigureerde account-id.
- De volledige IRC-kanaalconfiguratie (host/poort/TLS/kanalen/toestaanlijsten/vermeldingspoort) is gedocumenteerd in [IRC](/nl/channels/irc).

### Meerdere accounts (alle kanalen)

Voer meerdere accounts per kanaal uit (elk met een eigen `accountId`):

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- `default` wordt gebruikt wanneer `accountId` is weggelaten (CLI + routering).
- Omgevingstokens zijn alleen van toepassing op het **standaardaccount**.
- Basisinstellingen voor kanalen zijn van toepassing op alle accounts, tenzij ze per account worden overschreven.
- Gebruik `bindings[].match.accountId` om elk account naar een andere agent te routeren.
- Als je via `openclaw channels add` (of kanaalonboarding) een niet-standaardaccount toevoegt terwijl je nog een configuratie met één account op het hoogste kanaalniveau gebruikt, promoveert OpenClaw eerst de accountgebonden waarden voor één account op het hoogste niveau naar de accountmap van het kanaal, zodat het oorspronkelijke account blijft werken. De meeste kanalen verplaatsen deze naar `channels.<channel>.accounts.default`; Matrix kan in plaats daarvan een bestaand overeenkomend benoemd/standaarddoel behouden.
- Bestaande bindingen die alleen het kanaal bevatten (geen `accountId`) blijven overeenkomen met het standaardaccount; accountgebonden bindingen blijven optioneel.
- `openclaw doctor --fix` herstelt ook gemengde structuren door accountgebonden waarden voor één account op het hoogste niveau te verplaatsen naar het gepromoveerde account dat voor dat kanaal is gekozen. De meeste kanalen gebruiken `accounts.default`; Matrix kan in plaats daarvan een bestaand overeenkomend benoemd/standaarddoel behouden.

### Andere Plugin-kanalen

Veel Plugin-kanalen worden geconfigureerd als `channels.<id>` en zijn gedocumenteerd op hun eigen kanaalpagina's (bijvoorbeeld Feishu, LINE, Nextcloud Talk, Nostr, QQ Bot, Synology Chat, Twitch en Zalo).
Bekijk de volledige kanaalindex: [Kanalen](/nl/channels).

### Vermeldingspoort voor groepschats

Voor groepsberichten geldt standaard dat een **vermelding vereist** is (metadatavermelding of veilige regex-patronen). Dit geldt voor groepschats in WhatsApp, Telegram, Discord, Google Chat en iMessage.

Zichtbare antwoorden worden afzonderlijk beheerd. Normale directe verzoeken in groepen, kanalen en interne WebChat worden standaard automatisch definitief bezorgd: de definitieve assistenttekst wordt via het verouderde pad voor zichtbare antwoorden geplaatst. Kies voor `messages.visibleReplies: "message_tool"` of `messages.groupChat.visibleReplies: "message_tool"` wanneer door het model opgestelde bronantwoorden pas mogen worden geplaatst nadat de agent `message(action=send)` heeft aangeroepen. Als het model een inhoudelijk definitief antwoord retourneert zonder de berichtentool aan te roepen in een ingeschakelde modus die uitsluitend tools gebruikt, blijft die definitieve tekst privé, registreert het uitgebreide Gateway-logboek metadata van de onderdrukte payload en plaatst OpenClaw één herstelpoging in de wachtrij waarin het model wordt gevraagd hetzelfde antwoord via `message(action=send)` te bezorgen.

Het beleid dat uitsluitend tools gebruikt, regelt bronantwoorden van de assistent en generieke toolmedia. Het onderdrukt geen runtime-eigen terminaluitvoer, zoals geautoriseerde opdrachtantwoorden, permanente voltooiingsmeldingen of provider-eigen artefacten die door de beherende harness expliciet als host-eigendom zijn geclassificeerd. Artefacten die eigendom van de host zijn, worden via het normale kanaalverzendpad bezorgd en respecteren nog steeds de uitgaande weigering van `sendPolicy`. Omgevingsbeurten van `room_event` blijven stil tenzij het expliciete opdrachten zijn, zelfs wanneer runtime-uitvoer als host-eigendom is gemarkeerd.

Zichtbare antwoorden die uitsluitend tools gebruiken, vereisen een model/runtime dat betrouwbaar tools aanroept en worden aanbevolen voor gedeelde omgevingsruimten op modellen van de nieuwste generatie, zoals GPT-5.6 Sol. Sommige zwakkere modellen kunnen definitieve tekst als antwoord geven, maar begrijpen niet dat voor de bron zichtbare uitvoer met `message(action=send)` moet worden verzonden. OpenClaw herstelt het veelvoorkomende geval van een gestrand definitief antwoord standaard alleen wanneer het definitieve antwoord inhoudelijk is, de bronbeurt geen ruimtegebeurtenis was, het verzendbeleid de bezorging niet weigerde en er nog geen bronantwoord was verzonden. Herstel is beperkt tot één nieuwe poging; de synthetische prompt voor de nieuwe poging wordt niet opgeslagen en die poging wordt buiten collect-batching gehouden, zodat deze niet kan worden samengevoegd met niet-gerelateerde prompts in de wachtrij. Als de nieuwe poging eveneens strandt of niet in de wachtrij kan worden geplaatst, bezorgt OpenClaw alleen een opgeschoonde diagnose, zoals "Ik heb een antwoord gegenereerd, maar kon het niet in deze chat bezorgen. Probeer het opnieuw." De oorspronkelijke definitieve privétekst wordt nooit gemarkeerd voor automatische bezorging aan de bron. Gebruik voor modellen die herhaaldelijk antwoorden laten stranden `"automatic"`, zodat de definitieve assistentbeurt het zichtbare antwoordpad vormt, schakel over naar een sterker model dat tools kan aanroepen, inspecteer het uitgebreide Gateway-logboek op het overzicht van de onderdrukte payload of stel `messages.groupChat.visibleReplies: "automatic"` in om zichtbare definitieve antwoorden te gebruiken voor elk groeps-/kanaalverzoek.

Als de berichtentool niet beschikbaar is onder het actieve toolbeleid, valt OpenClaw terug op automatische zichtbare antwoorden in plaats van het antwoord stilzwijgend te onderdrukken. `openclaw doctor` waarschuwt voor deze inconsistentie.

Deze regel is van toepassing op normale definitieve agenttekst. Gespreksbindingen die eigendom zijn van een Plugin, gebruiken het door de eigenaar-Plugin geretourneerde antwoord als het zichtbare antwoord voor geclaimde beurten in gebonden threads; de Plugin hoeft voor die bindingsantwoorden `message(action=send)` niet aan te roepen.

**Probleemoplossing: @vermelding in groep activeert typen en daarna stilte (geen fout)**

Symptoom: een @vermelding in een groep/kanaal toont de typindicator en het Gateway-log meldt `dispatch complete (queuedFinal=false, replies=0)`, maar er verschijnt geen bericht in de ruimte. Privéberichten aan dezelfde agent krijgen normaal antwoord.

Oorzaak: de modus voor zichtbare antwoorden in de groep/het kanaal wordt omgezet naar `"message_tool"`, waardoor OpenClaw de beurt uitvoert maar de definitieve assistenttekst onderdrukt, tenzij de agent `message(action=send)` aanroept. In deze modus is er geen `NO_REPLY`-contract; zonder aanroep van de berichtentool is de oorspronkelijke definitieve tekst privé. Voor inhoudelijke bronbeurten probeert OpenClaw nu één beveiligde herstelpoging; korte notities, expliciete stilte, ruimtegebeurtenissen, beurten die door het verzendbeleid zijn geweigerd en reeds afgeleverde beurten worden niet opnieuw geprobeerd. Normale groeps- en kanaalbeurten gebruiken standaard `"automatic"`, dus dit symptoom treedt alleen op wanneer `messages.groupChat.visibleReplies` (of globaal `messages.visibleReplies`) expliciet is ingesteld op `"message_tool"`. Harness-`defaultVisibleReplies` is hier niet van toepassing — de resolver voor groepen/kanalen negeert dit; het is alleen van invloed op directe/brongesprekken (de Codex-harness onderdrukt op die manier definitieve antwoorden in directe gesprekken).

Oplossing: kies een model dat betrouwbaarder tools aanroept, verwijder de expliciete `"message_tool"`-overschrijving om terug te vallen op de standaardwaarde `"automatic"`, of stel `messages.groupChat.visibleReplies: "automatic"` in om zichtbare antwoorden voor elk groeps-/kanaalverzoek af te dwingen. Een inhoudelijk definitief antwoord dat niet kon worden afgeleverd, hoort niet langer als stilzwijgend succes te eindigen; het hoort te herstellen via één `message(action=send)`-poging of de opgeschoonde diagnostische melding voor een afleveringsfout te tonen. De Gateway laadt de `messages`-configuratie direct opnieuw nadat het bestand is opgeslagen; start de Gateway alleen opnieuw wanneer bestandsbewaking of het opnieuw laden van de configuratie in de implementatie is uitgeschakeld.

**Typen vermeldingen:**

- **Metadatavermeldingen**: systeemeigen @-vermeldingen van het platform. Genegeerd in de zelfchatmodus van WhatsApp.
- **Tekstpatronen**: veilige regexpatronen in `agents.entries.*.groupChat.mentionPatterns`. Ongeldige patronen en onveilige geneste herhaling worden genegeerd.
- Vermeldingscontrole wordt alleen afgedwongen wanneer detectie mogelijk is (systeemeigen vermeldingen of ten minste één patroon).

```json5
{
  messages: {
    visibleReplies: "automatic", // dwing oude automatische definitieve antwoorden af voor directe/brongesprekken
    groupChat: {
      historyLimit: 50,
      unmentionedInbound: "room_event", // altijd actieve niet-vermelde gesprekken in de ruimte worden stille context
      visibleReplies: "message_tool", // opt-in; vereist message(action=send) voor zichtbare antwoorden in de ruimte
    },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` stelt de globale standaardwaarde in. Kanalen kunnen dit overschrijven met `channels.<channel>.historyLimit` (of per account). Stel `0` in om dit uit te schakelen.

`messages.groupChat.unmentionedInbound: "room_event"` dient niet-vermelde, altijd actieve groeps-/kanaalberichten op ondersteunde kanalen in als stille ruimtecontext. Vermelde berichten, opdrachten en directe berichten blijven gebruikersverzoeken. Zie [Omgevingsgebeurtenissen in ruimten](/nl/channels/ambient-room-events) voor volledige voorbeelden voor Discord, Slack en Telegram.

`messages.visibleReplies` is de globale standaardwaarde voor brongebeurtenissen; `messages.groupChat.visibleReplies` overschrijft deze voor groeps-/kanaalbrongebeurtenissen. Wanneer `messages.visibleReplies` niet is ingesteld, gebruiken directe/brongesprekken de geselecteerde standaardwaarde van de runtime of harness, maar interne directe WebChat-beurten gebruiken automatische definitieve aflevering voor overeenstemming tussen Pi- en Codex-prompts. Stel `messages.visibleReplies: "message_tool"` in om bewust `message(action=send)` te vereisen voor zichtbare uitvoer. Toegestane lijsten voor kanalen en vermeldingscontrole bepalen nog steeds of een gebeurtenis wordt verwerkt.

#### Limieten voor de geschiedenis van privéberichten

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

Resolutie: overschrijving per privégesprek → standaardwaarde van provider → geen limiet (alles blijft behouden).

Deze resolver leest `channels.<provider>.dmHistoryLimit` en `channels.<provider>.dms.<id>.historyLimit` voor elk kanaal waarvan de sessiesleutel de standaardvorm `provider:direct:<id>` (of de verouderde vorm `provider:dm:<id>`) volgt. Daardoor werkt deze voor zowel gebundelde als Plugin-kanalen, niet alleen voor een vaste lijst.

#### Zelfchatmodus

Neem je eigen nummer op in `allowFrom` om de zelfchatmodus in te schakelen (negeert systeemeigen @-vermeldingen en reageert alleen op tekstpatronen):

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### Opdrachten (verwerking van chatopdrachten)

```json5
{
  commands: {
    native: "auto", // registreer systeemeigen opdrachten wanneer ondersteund
    nativeSkills: "auto", // registreer systeemeigen Skills-opdrachten wanneer ondersteund
    text: true, // parseer /opdrachten in chatberichten
    bash: false, // sta ! toe (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // sta /config toe
    mcp: false, // sta /mcp toe
    plugins: false, // sta /plugins toe
    debug: false, // sta /debug toe
    restart: true, // sta /restart + externe SIGUSR1-herstartverzoeken toe
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Opdrachtdetails">

- Dit blok configureert opdrachtinterfaces. Zie [Slash-opdrachten](/nl/tools/slash-commands) voor de huidige ingebouwde en gebundelde opdrachtencatalogus.
- Deze pagina is een **naslagwerk voor configuratiesleutels**, niet de volledige opdrachtencatalogus. Opdrachten die eigendom zijn van kanalen/Plugins, zoals QQ Bot `/bot-ping` `/bot-help` `/bot-logs`, LINE `/card`, apparaatkoppeling `/pair`, geheugen `/dreaming`, telefoonbediening `/phone` en Talk `/voice`, worden beschreven op hun kanaal-/Plugin-pagina's en in [Slash-opdrachten](/nl/tools/slash-commands).
- Tekstopdrachten moeten **zelfstandige** berichten zijn die beginnen met `/`.
- `native: "auto"` schakelt systeemeigen opdrachten in voor Discord/Telegram en laat ze uitgeschakeld voor Slack.
- `nativeSkills: "auto"` schakelt systeemeigen Skills-opdrachten in voor Discord/Telegram en laat ze uitgeschakeld voor Slack.
- Overschrijf dit per kanaal met `channels.discord.commands.native` (booleaanse waarde of `"auto"`). Voor Discord slaat `false` tijdens het opstarten de registratie en opschoning van systeemeigen opdrachten over.
- Overschrijf de registratie van systeemeigen Skills per kanaal met `channels.<provider>.commands.nativeSkills`.
- `channels.telegram.customCommands` voegt extra menu-items voor Telegram-bots toe.
- `bash: true` schakelt `! <cmd>` in voor de shell van de host. Vereist `tools.elevated.enabled` en dat de afzender in `tools.elevated.allowFrom.<channel>` staat.
- `config: true` schakelt `/config` in (leest/schrijft `openclaw.json`). Voor Gateway-`chat.send`-clients vereisen persistente schrijfbewerkingen met `/config set|unset` ook `operator.admin`; alleen-lezen `/config show` blijft beschikbaar voor normale operatorclients met schrijfrechten.
- `mcp: true` schakelt `/mcp` in voor door OpenClaw beheerde MCP-serverconfiguratie onder `mcp.servers`.
- `plugins: true` schakelt `/plugins` in voor het ontdekken, installeren en in-/uitschakelen van Plugins.
- `channels.<provider>.configWrites` beheert configuratiewijzigingen per kanaal (standaard: true).
- Voor kanalen met meerdere accounts beheert `channels.<provider>.accounts.<id>.configWrites` ook schrijfbewerkingen die op dat account zijn gericht (bijvoorbeeld `/allowlist --config --account <id>` of `/config set channels.<provider>.accounts.<id>...`).
- `restart: false` schakelt `/restart` en externe `SIGUSR1`-herstartverzoeken uit. Standaard: `true`.
- `ownerAllowFrom` is de expliciete lijst met toegestane eigenaars voor opdrachten die alleen voor eigenaars beschikbaar zijn en kanaalacties die door eigenaars worden beheerd. Deze staat los van `allowFrom`.
- `ownerDisplay: "hash"` hasht eigenaar-id's in de systeemprompt. Stel `ownerDisplaySecret` in om het hashen te beheren.
- `allowFrom` geldt per provider. Wanneer dit is ingesteld, is het de **enige** autorisatiebron (toegestane lijsten/koppelingen voor kanalen en `useAccessGroups` worden genegeerd).
- `useAccessGroups: false` staat toe dat opdrachten het beleid voor toegangsgroepen omzeilen wanneer `allowFrom` niet is ingesteld.
- Overzicht van documentatie over opdrachten:
  - ingebouwde en gebundelde catalogus: [Slash-opdrachten](/nl/tools/slash-commands)
  - kanaalspecifieke opdrachtinterfaces: [Kanalen](/nl/channels)
  - QQ Bot-opdrachten: [QQ Bot](/nl/channels/qqbot)
  - koppelingsopdrachten: [Koppeling](/nl/channels/pairing)
  - LINE-kaartopdracht: [LINE](/nl/channels/line)
  - geheugendromen: [Dreaming](/nl/concepts/dreaming)

</Accordion>

---

## Gerelateerd

- [Configuratienaslag](/nl/gateway/configuration-reference) — sleutels op het hoogste niveau
- [Configuratie — agents](/nl/gateway/config-agents)
- [Overzicht van kanalen](/nl/channels)
