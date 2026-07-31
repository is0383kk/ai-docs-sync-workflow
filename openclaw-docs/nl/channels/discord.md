---
read_when:
    - Werken aan functies voor Discord-kanalen
summary: Discord-botinstelling, configuratiesleutels, componenten, spraak en probleemoplossing
title: Discord
x-i18n:
    generated_at: "2026-07-27T05:33:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52a2926217f3a8dfb9398551ddacb0bc6aae6de0a164b215c55256eda9b6245e
    source_path: channels/discord.md
    workflow: 16
---

OpenClaw maakt via de officiële Discord-gateway verbinding met Discord als bot. DM's en guildkanalen worden ondersteund.

<CardGroup cols={3}>
  <Card title="Koppelen" icon="link" href="/nl/channels/pairing">
    Discord-DM's gebruiken standaard de koppelmodus.
  </Card>
  <Card title="Slash-opdrachten" icon="terminal" href="/nl/tools/slash-commands">
    Gedrag van native opdrachten en opdrachtencatalogus.
  </Card>
  <Card title="Problemen met kanalen oplossen" icon="wrench" href="/nl/channels/troubleshooting">
    Diagnose- en herstelprocedure voor meerdere kanalen.
  </Card>
</CardGroup>

## Snel instellen

Maak een Discord-applicatie met een bot, voeg de bot toe aan je server en koppel deze aan OpenClaw. Gebruik indien mogelijk een privéserver; [maak er zo nodig eerst een](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (**Create My Own > For me and my friends**).

<Steps>
  <Step title="Een Discord-applicatie en bot maken">
    Klik in de [Discord Developer Portal](https://discord.com/developers/applications) op **New Application** en geef de applicatie een naam (bijvoorbeeld "OpenClaw").

    Open **Bot** in de zijbalk en stel **Username** in op de naam van je agent.

  </Step>

  <Step title="Bevoorrechte intents inschakelen">
    Schakel, nog steeds op de pagina **Bot**, onder **Privileged Gateway Intents** het volgende in:

    - **Message Content Intent** (vereist)
    - **Server Members Intent** (aanbevolen; vereist voor roltoelatingslijsten, het koppelen van namen aan ID's en toegangsgroepen voor het kanaalpubliek)
    - **Presence Intent** (optioneel; alleen voor aanwezigheidsupdates)

  </Step>

  <Step title="Je bottoken kopiëren">
    Klik op de pagina **Bot** op **Reset Token** en kopieer het token.

    <Note>
    Ondanks de naam genereert dit je eerste token — er wordt niets 'opnieuw ingesteld'.
    </Note>

  </Step>

  <Step title="Een uitnodigings-URL genereren en de bot aan je server toevoegen">
    Open **OAuth2** in de zijbalk. Schakel in de **OAuth2 URL Generator** de volgende scopes in:

    - `bot`
    - `applications.commands`

    Schakel in het gedeelte **Bot Permissions** dat verschijnt ten minste het volgende in:

    **General Permissions**
      - View Channels

    **Text Permissions**
      - Send Messages
      - Read Message History
      - Embed Links
      - Attach Files
      - Add Reactions (optioneel)

    Dit is de basis voor normale tekstkanalen. Als de bot berichten in threads plaatst — waaronder workflows voor forum- of mediakanalen die een thread maken of voortzetten — schakel dan ook **Send Messages in Threads** in.

    Kopieer de gegenereerde URL, open deze in een browser, selecteer je server en klik op **Continue**. De bot hoort nu op je server te verschijnen.

  </Step>

  <Step title="Developer Mode inschakelen en je ID's verzamelen">
    Schakel Developer Mode in de Discord-app in, zodat je ID's kunt kopiëren:

    1. **User Settings** (tandwielpictogram) → **Developer** → schakel **Developer Mode** in
       *(op mobiel: **App Settings** → **Advanced**)*
    2. Klik met de rechtermuisknop op je **serverpictogram** → **Copy Server ID**
    3. Klik met de rechtermuisknop op je **eigen avatar** → **Copy User ID**

    Bewaar de Server ID en User ID samen met je bottoken; je hebt ze alle drie nodig voor de volgende stap.

  </Step>

  <Step title="DM's van serverleden toestaan">
    Voor het koppelen moet Discord toestaan dat de bot je een DM stuurt. Klik met de rechtermuisknop op je **serverpictogram** → **Privacy Settings** → schakel **Direct Messages** in.

    Laat dit ingeschakeld als je Discord-DM's met OpenClaw gebruikt. Als je alleen guildkanalen gebruikt, kun je het na het koppelen uitschakelen.

  </Step>

  <Step title="Je bottoken veilig instellen (stuur het niet via de chat)">
    Het bottoken is een geheim. Stel het in op de machine waarop OpenClaw wordt uitgevoerd voordat je je agent een bericht stuurt:

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
cat > discord.patch.json5 <<'JSON5'
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./discord.patch.json5 --dry-run
openclaw config patch --file ./discord.patch.json5
openclaw gateway
```

    Als OpenClaw al als achtergrondservice wordt uitgevoerd, start je deze opnieuw via de OpenClaw Mac-app of door het proces `openclaw gateway run` te stoppen en opnieuw te starten.
    Voer voor beheerde service-installaties `openclaw gateway install` uit vanuit een shell waarin `DISCORD_BOT_TOKEN` is ingesteld, of sla de variabele op in `~/.openclaw/.env`, zodat de service na het opnieuw starten de SecretRef uit de omgeving kan omzetten.
    Als je host wordt geblokkeerd of beperkt door Discords opzoekactie voor de applicatie bij het opstarten, stel je de applicatie-/client-ID uit de Developer Portal in, zodat die REST-aanroep bij het opstarten kan worden overgeslagen: `channels.discord.applicationId` voor het standaardaccount of `channels.discord.accounts.<accountId>.applicationId` per bot.

  </Step>

  <Step title="OpenClaw configureren en koppelen">

    <Tabs>
      <Tab title="Vraag het je agent">
        Chat met je OpenClaw-agent via een bestaand kanaal (bijvoorbeeld Telegram) en geef de instructie door. Als Discord je eerste kanaal is, gebruik je in plaats daarvan het tabblad CLI / configuratie.

        > "Ik heb mijn Discord-bottoken al in de configuratie ingesteld. Rond de Discord-configuratie af met User ID `<user_id>` en Server ID `<server_id>`."
      </Tab>
      <Tab title="CLI / configuratie">
        Configuratie via een bestand:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Terugval op een omgevingsvariabele voor het standaardaccount:

```bash
DISCORD_BOT_TOKEN=...
```

        Schrijf voor een gescripte of externe configuratie hetzelfde JSON5-blok met `openclaw config patch --file ./discord.patch.json5 --dry-run` en voer het daarna opnieuw uit zonder `--dry-run`. Platte-tekenreeksen voor `token` werken ook en SecretRef-waarden worden voor `channels.discord.token` ondersteund via env-/file-/exec-providers. Zie [Geheimenbeheer](/nl/gateway/secrets).

        Bewaar bij meerdere Discord-bots het bottoken en de applicatie-ID van elke bot onder het bijbehorende account. Een `channels.discord.applicationId` op het hoogste niveau wordt door accounts overgenomen; stel deze daar dus alleen in wanneer elk account dezelfde applicatie-ID gebruikt.

```json5
{
  channels: {
    discord: {
      enabled: true,
      accounts: {
        personal: {
          token: { source: "env", provider: "default", id: "DISCORD_PERSONAL_TOKEN" },
          applicationId: "111111111111111111",
        },
        work: {
          token: { source: "env", provider: "default", id: "DISCORD_WORK_TOKEN" },
          applicationId: "222222222222222222",
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="De eerste DM-koppeling goedkeuren">
    Stuur je bot een DM in Discord zodra de gateway actief is. De bot antwoordt met een koppelcode.

    <Tabs>
      <Tab title="Vraag het je agent">
        Stuur de koppelcode via je bestaande kanaal naar je agent:

        > "Keur deze Discord-koppelcode goed: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    Koppelcodes verlopen na 1 uur. Na goedkeuring kun je via een Discord-DM met je agent chatten.

  </Step>
</Steps>

<Note>
Het omzetten van tokens houdt rekening met accounts. Tokenwaarden uit de configuratie hebben voorrang op de terugval via de omgevingsvariabele en `DISCORD_BOT_TOKEN` wordt alleen voor het standaardaccount gebruikt.
Als twee ingeschakelde Discord-accounts hetzelfde bottoken opleveren, start OpenClaw slechts één gatewaymonitor voor dat token: een token uit de configuratie heeft voorrang op de terugval via de omgevingsvariabele; anders krijgt het eerste ingeschakelde account voorrang en wordt het dubbele account als uitgeschakeld gemeld met reden `duplicate bot token`.
Voor geavanceerde uitgaande aanroepen (berichttool/kanaalacties) wordt een expliciete `token` per aanroep voor die aanroep gebruikt. Dit geldt voor verzendacties en acties in de stijl van lezen/controleren (lezen/zoeken/ophalen/thread/vastgemaakte berichten/machtigingen). Het accountbeleid en de instellingen voor nieuwe pogingen zijn nog steeds afkomstig van het geselecteerde account in de actieve runtime-snapshot.
</Note>

## Aanbevolen: een guildwerkruimte instellen

Zodra DM's werken, kun je je server omvormen tot een volledige werkruimte waarin elk kanaal een eigen agentsessie met een eigen context krijgt. Aanbevolen voor privéservers waarop alleen jij en je bot aanwezig zijn.

<Steps>
  <Step title="Je server aan de guildtoelatingslijst toevoegen">
    Hierdoor kan je agent in elk kanaal op je server antwoorden, niet alleen in DM's.

    <Tabs>
      <Tab title="Vraag het je agent">
        > "Voeg mijn Discord Server ID `<server_id>` toe aan de guildtoelatingslijst"
      </Tab>
      <Tab title="Configuratie">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Antwoorden zonder @vermelding toestaan">
    Standaard antwoordt de agent in guildkanalen alleen wanneer deze met een @vermelding wordt genoemd. Op een privéserver wil je waarschijnlijk dat de agent op elk bericht antwoordt.

    In guildkanalen worden normale antwoorden standaard automatisch geplaatst. Schakel voor gedeelde ruimtes die altijd actief zijn `messages.groupChat.visibleReplies: "message_tool"` in, zodat de agent kan meelezen en alleen een bericht plaatst wanneer deze besluit dat een antwoord in het kanaal nuttig is. Dit werkt het beste met modellen van de nieuwste generatie die betrouwbaar met tools omgaan, zoals GPT-5.6 Sol. Omgevingsgebeurtenissen in ruimtes blijven stil tenzij de tool iets verzendt. Zie [Omgevingsgebeurtenissen in ruimtes](/nl/channels/ambient-room-events) voor de volledige configuratie van de meeleesmodus.

    Als Discord aangeeft dat er wordt getypt en de logboeken tokengebruik tonen, maar er geen bericht wordt geplaatst, controleer dan of de beurt als omgevingsgebeurtenis in een ruimte was geconfigureerd of was ingesteld op zichtbare antwoorden via de berichttool.

    <Tabs>
      <Tab title="Vraag het je agent">
        > "Sta toe dat mijn agent op deze server antwoordt zonder dat deze met een @vermelding hoeft te worden genoemd"
      </Tab>
      <Tab title="Configuratie">
        Stel `requireMention: false` in je guildconfiguratie in:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

        Stel `messages.groupChat.visibleReplies: "message_tool"` in om te vereisen dat zichtbare antwoorden in groepen/kanalen via de berichttool worden verzonden.

      </Tab>
    </Tabs>

  </Step>

  <Step title="Geheugen voor guildkanalen plannen">
    Langetermijngeheugen (MEMORY.md) wordt alleen automatisch in DM-sessies geladen; guildkanalen laden het niet.

    <Tabs>
      <Tab title="Vraag het je agent">
        > "Wanneer ik vragen stel in Discord-kanalen, gebruik je memory_search of memory_get als je langetermijncontext uit MEMORY.md nodig hebt."
      </Tab>
      <Tab title="Handmatig">
        Plaats voor gedeelde context in elk kanaal stabiele instructies in `AGENTS.md` of `USER.md` (wordt voor elke sessie geïnjecteerd). Bewaar langetermijnnotities in `MEMORY.md` en open ze wanneer nodig met geheugentools.
      </Tab>
    </Tabs>

  </Step>
</Steps>

Maak nu kanalen en begin te chatten. De agent ziet de kanaalnaam en elk kanaal is een geïsoleerde sessie — stel `#coding`, `#home`, `#research` in, of wat er maar bij je workflow past.

## Runtimemodel

- Gateway beheert de Discord-verbinding.
- De routering van antwoorden is deterministisch: inkomende Discord-berichten worden op Discord beantwoord.
- Metadata van Discord-guilds en -kanalen wordt als niet-vertrouwde context aan de modelprompt toegevoegd, niet als een voor de gebruiker zichtbaar antwoordvoorvoegsel. Als een model die envelop terugkopieert, verwijdert OpenClaw de gekopieerde metadata uit uitgaande antwoorden en uit toekomstige herhalingscontext.
- Standaard (`session.dmScope=main`) delen directe chats de hoofdsessie van de agent (`agent:main:main`).
- Guildkanalen hebben geïsoleerde sessiesleutels (`agent:<agentId>:discord:channel:<channelId>`).
- Groeps-DM's worden standaard genegeerd (`channels.discord.dm.groupEnabled=false`).
- Native slash-opdrachten worden uitgevoerd in geïsoleerde opdrachtsessies (`agent:<agentId>:discord:slash:<userId>`), waarbij `CommandTargetSessionKey` nog steeds wordt doorgegeven aan de gerouteerde gesprekssessie.
- De levering van aankondigingen van Cron/Heartbeat met alleen tekst aan Discord wordt samengevoegd tot het uiteindelijke, voor de assistent zichtbare antwoord en eenmaal verzonden. Media en gestructureerde componentpayloads blijven uit meerdere berichten bestaan wanneer de agent meerdere leverbare payloads produceert.

## Forumkanalen

Discord-forum- en mediakanalen accepteren alleen berichten in threads. OpenClaw ondersteunt twee manieren om deze te maken:

- Stuur een bericht naar het bovenliggende forumkanaal (`channel:<forumId>`) om automatisch een thread te maken. De threadtitel is de eerste niet-lege regel van het bericht (afgekapt op Discords limiet van 100 tekens voor threadnamen).
- Gebruik `openclaw message thread create` om rechtstreeks een thread te maken. Geef `--message-id` niet door voor forumkanalen.

Stuur naar het bovenliggende forumkanaal om een thread te maken:

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Onderwerptitel\nInhoud van het bericht"
```

Maak expliciet een forumthread:

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Onderwerptitel" --message "Inhoud van het bericht"
```

Bovenliggende forumkanalen accepteren geen Discord-componenten. Als je componenten nodig hebt, stuur je het bericht naar de thread zelf (`channel:<threadId>`).

## Interactieve componenten

OpenClaw ondersteunt containers met Discord-componenten v2 voor agentberichten. Gebruik de berichttool met een `components`-payload. Interactieresultaten worden als normale inkomende berichten teruggestuurd naar de agent en volgen de bestaande Discord-instellingen voor `replyToMode`.

Ondersteunde blokken:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Actierijen staan maximaal 5 knoppen of één keuzemenu toe
- Selectietypen: `string`, `user`, `role`, `mentionable`, `channel`

Componenten zijn standaard eenmalig te gebruiken. Stel `components.reusable=true` in om knoppen, keuzemenu's en formulieren meerdere keren te kunnen gebruiken totdat ze verlopen.

Om te beperken wie op een knop kan klikken, stel je `allowedUsers` in voor die knop (Discord-gebruikers-ID's, tags of `*`). Niet-overeenkomende gebruikers ontvangen een tijdelijke weigering die alleen voor hen zichtbaar is.

Componentcallbacks verlopen standaard na 30 minuten. Stel `channels.discord.agentComponents.ttlMs` in om de levensduur van het callbackregister voor het standaardaccount te wijzigen, of `channels.discord.accounts.<accountId>.agentComponents.ttlMs` per account. De waarde is in milliseconden, moet een positief geheel getal zijn en is begrensd op `86400000` (24 uur). Langere TTL's zijn geschikt voor review- en goedkeuringsworkflows waarin knoppen bruikbaar moeten blijven, maar verlengen de periode waarin een oud Discord-bericht nog een actie kan activeren. Gebruik bij voorkeur de kortste passende TTL en behoud de standaardwaarde wanneer verouderde callbacks onverwacht zouden zijn.

De slash-opdrachten `/model` en `/models` openen een interactieve modelkiezer met vervolgkeuzelijsten voor provider, model en compatibele runtime, plus een verzendstap. `/models add` is verouderd en retourneert een afschaffingsbericht in plaats van modellen vanuit de chat te registreren. Het antwoord van de kiezer is tijdelijk, alleen zichtbaar voor de gebruiker die deze heeft aangeroepen en alleen door die gebruiker te gebruiken. Discord-keuzemenu's zijn beperkt tot 25 opties. Voeg daarom `provider/*`-vermeldingen toe aan `agents.defaults.modelPolicy.allow` als je wilt dat de kiezer dynamisch ontdekte modellen alleen toont voor geselecteerde providers, zoals `openai` of `vllm`.

Bestandsbijlagen:

- `file`-blokken moeten naar een bijlageverwijzing wijzen (`attachment://<filename>`)
- Lever de bijlage via `media`/`path`/`filePath` (één bestand); gebruik `media-gallery` voor meerdere bestanden
- Gebruik `filename` om de uploadnaam te overschrijven wanneer deze met de bijlageverwijzing moet overeenkomen

Modale formulieren:

- Voeg `components.modal` toe met maximaal 5 velden
- Veldtypen: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw voegt automatisch een activeringsknop toe

Voorbeeld:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optionele reservetekst",
  components: {
    reusable: true,
    text: "Kies een pad",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Goedkeuren",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Afwijzen", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Kies een optie",
          options: [
            { label: "Optie A", value: "a" },
            { label: "Optie B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Formulier openen",
      fields: [
        { type: "text", label: "Aanvrager" },
        {
          type: "select",
          label: "Prioriteit",
          options: [
            { label: "Laag", value: "low" },
            { label: "Hoog", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Toegangsbeheer en routering

<Tabs>
  <Tab title="DM-beleid">
    `channels.discord.dmPolicy` beheert DM-toegang. `channels.discord.allowFrom` is de canonieke DM-toelatingslijst.

    - `pairing` (standaard)
    - `allowlist` (vereist ten minste één `allowFrom`-afzender)
    - `open` (vereist dat `channels.discord.allowFrom` `"*"` bevat)
    - `disabled`

    Als het DM-beleid niet open is, worden onbekende gebruikers geblokkeerd (of gevraagd om te koppelen in de modus `pairing`).

    Voorrangsregels voor meerdere accounts:

    - `channels.discord.accounts.default.allowFrom` geldt alleen voor het account `default`.
    - Voor één account heeft `allowFrom` voorrang op de verouderde `dm.allowFrom`.
    - Benoemde accounts nemen `channels.discord.allowFrom` over wanneer hun eigen `allowFrom` en de verouderde `dm.allowFrom` niet zijn ingesteld.
    - Benoemde accounts nemen `channels.discord.accounts.default.allowFrom` niet over.

    De verouderde `channels.discord.dm.policy` en `channels.discord.dm.allowFrom` worden voor compatibiliteit nog steeds gelezen. `openclaw doctor --fix` migreert deze naar `dmPolicy` en `allowFrom` wanneer dat mogelijk is zonder de toegang te wijzigen.

    DM-doelindeling voor bezorging:

    - `user:<id>`
    - `<@id>`-vermelding

    Kale numerieke ID's worden normaal als kanaal-ID's geïnterpreteerd wanneer een standaardkanaal actief is, maar ID's in de effectieve DM-`allowFrom` van het account worden voor compatibiliteit behandeld als DM-doelen voor gebruikers.

  </Tab>

  <Tab title="Toegangsgroepen">
    Discord-DM's en autorisatie voor tekstopdrachten kunnen dynamische `accessGroup:<name>`-vermeldingen in `channels.discord.allowFrom` gebruiken.

    Namen van toegangsgroepen worden gedeeld tussen berichtkanalen. Gebruik `type: "message.senders"` voor een statische groep waarvan de leden worden uitgedrukt met de normale `allowFrom`-syntaxis van elk kanaal, of `type: "discord.channelAudience"` wanneer de huidige `ViewChannel`-doelgroep van een Discord-kanaal het lidmaatschap dynamisch moet bepalen. Gedeeld gedrag van toegangsgroepen: [Toegangsgroepen](/nl/channels/access-groups).

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

    Een Discord-tekstkanaal heeft geen afzonderlijke ledenlijst. `type: "discord.channelAudience"` modelleert lidmaatschap als volgt: de DM-afzender is lid van de geconfigureerde server en heeft momenteel effectieve `ViewChannel`-toestemming voor het geconfigureerde kanaal nadat rol- en kanaaloverschrijvingen zijn toegepast.

    Voorbeeld: sta iedereen die `#maintainers` kan zien toe om de bot een DM te sturen, terwijl DM's voor alle anderen gesloten blijven.

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

    Je kunt dynamische en statische vermeldingen combineren:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers", "discord:123456789012345678"],
    },
  },
}
```

    Zoekopdrachten weigeren bij fouten de toegang. Als Discord `Missing Access` retourneert, het opzoeken van het lid mislukt of het kanaal bij een andere server hoort, wordt de DM-afzender als niet-geautoriseerd behandeld.

    Schakel in de Discord Developer Portal **Server Members Intent** in wanneer je toegangsgroepen op basis van een kanaaldoelgroep gebruikt. DM's bevatten geen lidstatus van de server, dus OpenClaw zoekt het lid tijdens de autorisatie op via Discord REST.

  </Tab>

  <Tab title="Serverbeleid">
    De verwerking van servers wordt beheerd door `channels.discord.groupPolicy`:

    - `open`
    - `allowlist`
    - `disabled`

    De veilige basisinstelling wanneer `channels.discord` bestaat, is `allowlist`.

    Gedrag van `allowlist`:

    - de server moet overeenkomen met `channels.discord.guilds` (`id` heeft de voorkeur, slug wordt geaccepteerd)
    - optionele toelatingslijsten voor afzenders: `users` (stabiele ID's aanbevolen) en `roles` (alleen rol-ID's); als een van beide is geconfigureerd, zijn afzenders toegestaan wanneer ze overeenkomen met `users` OF `roles`
    - rechtstreekse overeenkomst op naam/tag is standaard uitgeschakeld; schakel `channels.discord.dangerouslyAllowNameMatching: true` alleen in als noodcompatibiliteitsmodus
    - namen/tags worden ondersteund voor `users`, maar ID's zijn veiliger; `openclaw security audit` waarschuwt wanneer vermeldingen met namen/tags worden gebruikt
    - als voor een server `channels` is geconfigureerd, worden niet-vermelde kanalen geweigerd
    - als een server geen `channels`-blok heeft, zijn alle kanalen in die toegestane server toegestaan

    Voorbeeld:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { enabled: true },
            help: { enabled: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    De verouderde sleutel `allow` per kanaal wordt door `openclaw doctor --fix` gemigreerd naar `enabled`.

    Als je alleen `DISCORD_BOT_TOKEN` instelt en geen `channels.discord`-blok maakt, is de runtime-terugval `groupPolicy="allowlist"` (met een waarschuwing in de logboeken), zelfs als `channels.defaults.groupPolicy` `open` is.

  </Tab>

  <Tab title="Vermeldingen en groeps-DM's">
    Berichten op servers vereisen standaard een vermelding.

    Detectie van vermeldingen omvat:

    - expliciete vermelding van de bot
    - geconfigureerde vermeldingspatronen (`agents.entries.*.groupChat.mentionPatterns`, met `messages.groupChat.mentionPatterns` als terugval)
    - impliciet antwoord-aan-bot-gedrag in ondersteunde gevallen

    Gebruik bij het schrijven van uitgaande Discord-berichten de canonieke vermeldingssyntaxis: `<@USER_ID>` voor gebruikers, `<#CHANNEL_ID>` voor kanalen en `<@&ROLE_ID>` voor rollen. Gebruik niet de verouderde vorm `<@!USER_ID>` voor vermeldingen met een bijnaam.

    `requireMention` wordt per server/kanaal geconfigureerd (`channels.discord.guilds...`).
    `ignoreOtherMentions` negeert optioneel berichten die een andere gebruiker/rol vermelden, maar niet de bot (met uitzondering van @everyone/@here).

    Groeps-DM's:

    - standaard: genegeerd (`dm.groupEnabled=false`)
    - optionele toelatingslijst via `dm.groupChannels` (kanaal-ID's of slugs)

  </Tab>
</Tabs>

### Agentroutering op basis van rollen

Gebruik `bindings[].match.roles` om leden van Discord-servers op basis van hun rol-ID naar verschillende agents te routeren. Rolgebaseerde bindingen accepteren alleen rol-ID's en worden geëvalueerd na peer- of bovenliggende-peerbindingen en vóór bindingen die alleen voor een server gelden. Als een binding ook andere overeenkomstvelden instelt (bijvoorbeeld `peer` + `guildId` + `roles`), moeten alle geconfigureerde velden overeenkomen.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Systeemeigen opdrachten en opdrachtauthenticatie

- `commands.native` is standaard `"auto"` en is ingeschakeld voor Discord.
- Overschrijving per kanaal: `channels.discord.commands.native`.
- `commands.native=false` slaat de registratie en opschoning van Discord-slashopdrachten tijdens het opstarten over. Eerder geregistreerde opdrachten kunnen zichtbaar blijven in Discord totdat je ze uit de Discord-app verwijdert.
- Authenticatie van systeemeigen opdrachten gebruikt dezelfde Discord-toelatingslijsten en hetzelfde beleid als normale berichtverwerking.
- Opdrachten kunnen in de Discord-interface nog steeds zichtbaar zijn voor onbevoegde gebruikers; bij uitvoering wordt OpenClaw-authenticatie afgedwongen en wordt "niet geautoriseerd" geantwoord.
- Standaardinstellingen voor slashopdrachten: `ephemeral: true` (`channels.discord.slashCommand.ephemeral`).

Zie [Slashopdrachten](/nl/tools/slash-commands) voor de opdrachtencatalogus en het gedrag.

## Functiedetails

<AccordionGroup>
  <Accordion title="Antwoordtags en systeemeigen antwoorden">
    Discord ondersteunt antwoordtags in agentuitvoer:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    Aangestuurd door `channels.discord.replyToMode`:

    - `off` (standaard): geen impliciete antwoordthreads; expliciete `[[reply_to_*]]`-tags worden nog steeds gehonoreerd
    - `first`: koppelt de impliciete systeemeigen antwoordverwijzing aan het eerste uitgaande Discord-bericht van de beurt
    - `all`: koppelt deze aan elk uitgaand bericht
    - `batched`: koppelt deze alleen wanneer de inkomende gebeurtenis een gedebouncete batch van meerdere berichten was — handig wanneer je systeemeigen antwoorden voornamelijk wilt gebruiken voor onduidelijke chats met plotselinge berichtenreeksen, en niet voor elke beurt met één bericht

    Bericht-ID's worden beschikbaar gesteld in de context en geschiedenis, zodat agents specifieke berichten kunnen adresseren.

  </Accordion>

  <Accordion title="Linkvoorbeelden">
    Discord genereert standaard uitgebreide linkinsluitingen voor URL's. OpenClaw onderdrukt deze gegenereerde insluitingen standaard in uitgaande Discord-berichten, zodat door agents verzonden URL's gewone links blijven, tenzij je dit inschakelt:

```json5
{
  channels: {
    discord: {
      suppressEmbeds: false,
    },
  },
}
```

    Stel `channels.discord.accounts.<id>.suppressEmbeds` in om dit voor één account te overschrijven. Verzendingen via de berichtentool van de agent kunnen ook `suppressEmbeds: false` doorgeven voor één bericht. Expliciete Discord-`embeds`-payloads worden niet onderdrukt door de standaardinstelling voor linkvoorbeelden.

  </Accordion>

  <Accordion title="Livestreamvoorbeeld">
    OpenClaw kan conceptantwoorden streamen door een tijdelijk bericht te verzenden en dit te bewerken naarmate tekst binnenkomt. `channels.discord.streaming.mode` accepteert `off` | `partial` | `block` | `progress` (standaard wanneer geen `streaming`- of verouderde `streamMode`-sleutel is ingesteld). `streamMode` is een verouderde alias; voer `openclaw doctor --fix` uit om opgeslagen configuratie te herschrijven naar de canonieke geneste `streaming`-structuur.

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: false,
          commentary: false,
        },
      },
    },
  },
}
```

    - `off` schakelt bewerkingen van Discord-voorbeelden uit.
    - `partial` bewerkt één voorbeeldbericht naarmate tokens binnenkomen.
    - `block` produceert brokken ter grootte van een concept; stel de grootte en afbreekpunten af met `streaming.preview.chunk` (`minChars`, `maxChars`, `breakPreference`), begrensd op `textChunkLimit`. Wanneer blokstreaming expliciet is ingeschakeld, slaat OpenClaw de voorbeeldstream over om dubbel streamen te voorkomen.
    - `progress` behoudt één bewerkbaar statusconcept tot de definitieve levering. Standaard toont het één regel met de recentste inleiding of toelichting van de agent, zonder gegenereerd label, witregel of toolrijen.
    - Media, fouten en definitieve expliciete antwoorden annuleren wachtende voorbeeldbewerkingen.
    - `streaming.preview.toolProgress` is standaard `true` in de modus `partial`/`block`. De voortgangsmodus van Discord toont standaard geen toolrijen; stel `streaming.progress.toolProgress: true` in om ze in te schakelen.
    - Stel `streaming.progress.toolProgress: true` in om compacte tool-/voortgangsrijen toe te voegen, zoals `🛠️ Bash: run tests` of `🔎 Web Search: for "query"`. Voor compatibiliteit behoudt een bestaande `progress.label`- of `progress.labels`-configuratie de eerdere standaard voor toolrijen; stel `toolProgress: false` in voor een aangepast label zonder rijen.
    - `streaming.progress.commentary` (standaard `false`) schakelt onbewerkte assistenttoelichting in het tijdelijke voortgangsconcept in. De standaardstatusregel met inleiding/toelichting staat los van deze optie. Toelichting wordt vóór weergave opgeschoond, blijft tijdelijk en verandert de levering van het definitieve antwoord niet.
    - `streaming.progress.maxLineChars` bepaalt het budget per regel voor het voortgangsvoorbeeld. Proza wordt op woordgrenzen ingekort; details van opdrachten en paden behouden nuttige achtervoegsels.
    - `streaming.preview.commandText` / `streaming.progress.commandText` bepaalt de details van opdrachten/uitvoering in compacte voortgangsregels: `raw` (standaard) of `status` (alleen het toollabel).

    Verberg onbewerkte opdracht-/uitvoeringstekst, maar behoud compacte voortgangsregels:

    ```json
    {
      "channels": {
        "discord": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    Voorbeeldstreaming ondersteunt alleen tekst; antwoorden met media vallen terug op normale levering.

  </Accordion>

  <Accordion title="Geschiedenis, context en threadgedrag">
    Geschiedeniscontext van de guild:

    - `channels.discord.historyLimit` standaard `20`
    - terugval: `messages.groupChat.historyLimit`
    - `0` schakelt dit uit

    Beheer van DM-geschiedenis:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    Threadgedrag:

    - Discord-threads worden gerouteerd als kanaalsessies en nemen de configuratie van het bovenliggende kanaal over, tenzij deze wordt overschreven.
    - Threadsessies nemen de `/model`-selectie op sessieniveau van het bovenliggende kanaal over als terugval die alleen voor het model geldt; threadlokale `/model`-selecties hebben voorrang en de transcriptgeschiedenis van het bovenliggende kanaal wordt alleen gekopieerd als transcriptovererving is ingeschakeld.
    - `channels.discord.thread.inheritParent` (standaard `false`) laat nieuwe automatische threads beginnen met gegevens uit het bovenliggende transcript. Overschrijving per account: `channels.discord.accounts.<id>.thread.inheritParent`.
    - Reacties van de berichtentool kunnen `user:<id>`-DM-doelen herleiden.
    - `guilds.<guild>.channels.<channel>.requireMention: false` blijft behouden tijdens terugval bij activering van de antwoordfase.

    Kanaalonderwerpen worden als **niet-vertrouwde** context geïnjecteerd. Toelatingslijsten bepalen wie de agent kan activeren, maar vormen geen volledige grens voor het redigeren van aanvullende context.

  </Accordion>

  <Accordion title="Aan threads gekoppelde sessies voor subagents">
    Discord kan een thread aan een sessiedoel koppelen, zodat vervolgberichten in die thread naar dezelfde sessie gerouteerd blijven worden (inclusief subagentsessies).

    Opdrachten:

    - `/focus <target>` koppelt de huidige/nieuwe thread aan een subagent-/sessiedoel
    - `/unfocus` verwijdert de huidige threadkoppeling
    - `/agents` toont actieve uitvoeringen en de koppelingsstatus
    - `/session idle <duration|off>` bekijkt/wijzigt het automatisch opheffen van de focus na inactiviteit voor gefocuste koppelingen
    - `/session max-age <duration|off>` bekijkt/wijzigt de harde maximumleeftijd voor gefocuste koppelingen

    Configuratie:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
      defaultSpawnContext: "fork",
    },
  },
}
```

    Opmerkingen:

    - `session.threadBindings.*` is het canonieke beleid voor Discord en Telegram.
    - `spawnSessions` bepaalt het automatisch maken/koppelen van threads voor `sessions_spawn({ thread: true })` en ACP-threadstarts. Standaard: `true`.
    - `defaultSpawnContext` bepaalt de systeemeigen subagentcontext voor aan threads gekoppelde starts. Standaard: `"fork"`.
    - Verouderde `spawnSubagentSessions`-/`spawnAcpSessions`-sleutels worden gemigreerd door `openclaw doctor --fix`.
    - Als threadkoppelingen zijn uitgeschakeld, zijn `/focus` en gerelateerde bewerkingen niet beschikbaar.

    Zie [Subagents](/nl/tools/subagents), [ACP-agents](/nl/tools/acp-agents) en [Configuratiereferentie](/nl/gateway/configuration-reference).

  </Accordion>

  <Accordion title="Voortgang van subagents in het bronbericht">
    Stel `channels.discord.subagentProgress: true` in om activiteit van onderliggende processen op de achtergrond weer te geven bij het Discord-bericht dat de bovenliggende uitvoering heeft gestart.

```json5
{
  channels: {
    discord: {
      subagentProgress: true,
    },
  },
}
```

    Terwijl onderliggende uitvoeringen actief zijn, houdt OpenClaw de type-indicator van Discord maximaal één uur actief en vervangt het één telreactie (`1️⃣` tot en met `🔟`) wanneer het aantal gelijktijdige uitvoeringen verandert; `🔟` staat ook voor 10 of meer. De telreactie wordt verwijderd nadat het laatste onderliggende proces is beëindigd. Een mislukt, verlopen of afgebroken onderliggend proces laat een `🔴`-reactie achter.

    Dit is optioneel en gebruikt vaste interne standaardwaarden voor timing en emoji. De bot heeft de machtiging **Add Reactions** nodig voor feedback via reacties. `channels.discord.accounts.<id>.subagentProgress` op accountniveau overschrijft de waarde op het hoogste niveau.

  </Accordion>

  <Accordion title="Permanente ACP-kanaalkoppelingen">
    Configureer voor stabiele ACP-werkruimten die "altijd actief" zijn getypeerde ACP-koppelingen op het hoogste niveau die op Discord-gesprekken zijn gericht.

    Configuratiepad: `bindings[]` met `type: "acp"` en `match.channel: "discord"`.

```json5
{
  agents: {
    entries: {
      codex: {
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
    },
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    Opmerkingen:

    - `/acp spawn codex --bind here` koppelt het huidige kanaal of de huidige thread ter plaatse en houdt toekomstige berichten in dezelfde ACP-sessie. Threadberichten nemen de koppeling van het bovenliggende kanaal over.
    - In een gekoppeld kanaal of een gekoppelde thread stellen `/new` en `/reset` dezelfde ACP-sessie ter plaatse opnieuw in. Tijdelijke threadkoppelingen kunnen de doelbepaling overschrijven zolang ze actief zijn.
    - `spawnSessions` beheert het maken/koppelen van onderliggende threads via `--thread auto|here`.

    Zie [ACP-agents](/nl/tools/acp-agents) voor details over het koppelingsgedrag.

  </Accordion>

  <Accordion title="Reactiemeldingen">
    Reactiemeldingsmodus per guild (`guilds.<id>.reactionNotifications`):

    - `off`
    - `own` (standaard)
    - `all`
    - `allowlist` (gebruikt `guilds.<id>.users`)

    Reactiegebeurtenissen worden omgezet in systeemgebeurtenissen en aan de gerouteerde Discord-sessie gekoppeld.

  </Accordion>

  <Accordion title="Online-aanwezigheidsgebeurtenissen">
    Laat een guild gerouteerde agentactiveringen ontvangen wanneer een menselijk lid van offline naar online overgaat:

    ```json5
    {
      channels: {
        discord: {
          intents: { presence: true },
          guilds: {
            "111111111111111111": {
              presenceEvents: {
                channelId: "222222222222222222",
                users: ["333333333333333333"], // optional; further narrow channel viewers
                reconnectSuppressSeconds: 300, // optional; new-session quiet window (0 disables)
                burstLimit: 8, // optional; max events per burst window
                burstWindowSeconds: 60, // optional; sliding burst-detection window
              },
            },
          },
        },
      },
    }
    ```

    `presenceEvents` vereist dat Heartbeat is ingeschakeld voor de gerouteerde agent en dat de bevoorrechte **Presence Intent** is ingeschakeld op de Bot-pagina van de toepassing in het Discord Developer Portal. OpenClaw vult de huidige online leden vanuit elke volledige `GUILD_CREATE`-momentopname, routeert waargenomen overgangen van offline naar online en beschouwt ook een later eerste onlinesignaal voor een niet eerder waargenomen lid als nieuw beschikbaar. Dat lid kan online zijn gekomen of na de momentopname zijn toegetreden, waardoor de gebeurtenis geen exacte eerdere status aangeeft. Alleen mensen die `channelId` kunnen bekijken, komen in aanmerking: voor kanalen en openbare threads is **View Channel** vereist voor het kanaal of bovenliggende kanaal, terwijl voor privéthreads daarnaast lidmaatschap of **Manage Threads** vereist is. `users` kan die doelgroep verder beperken. OpenClaw negeert bots en ongewijzigde onlinestatussen en bewaart een afkoelperiode van acht uur per gebruiker tussen herstarts van de Gateway. Wanneer Discord een nieuwe Gateway-sessie tot stand brengt en `READY` verzendt, onderdrukt OpenClaw van aanwezigheid afgeleide gebeurtenissen gedurende `reconnectSuppressSeconds` (standaard 300, `0` schakelt dit uit) terwijl de aanwezigheidsstatus van de guild opnieuw wordt opgebouwd, zodat opnieuw waargenomen leden de agent niet één voor één kunnen activeren. Daarnaast beperkt het systeem het aantal succesvol in de wachtrij geplaatste gebeurtenissen per guild tot `burstLimit` gebeurtenissen (standaard 8) per voortschrijdend venster van `burstWindowSeconds` (standaard 60), waarbij elke onderdrukkingsperiode van een guild eenmaal wordt geregistreerd. Een hervatte sessie wordt niet als een nieuwe sessie behandeld. Discord beperkt momentopnamen voor guilds met meer dan 75.000 leden; daar vereist OpenClaw een expliciete offline-update voordat een begroeting wordt verstuurd. De systeemgebeurtenis bevat onveranderlijke gebruikers-, guild- en kanaal-ID's zonder veranderlijke weergavenamen op te nemen. De agent beslist of en hoe er wordt begroet.

  </Accordion>

  <Accordion title="Bevestigingsreacties">
    `ackReaction` verzendt een bevestigingsemoji terwijl OpenClaw een inkomend bericht verwerkt.

    Volgorde van bepaling:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - terugval op de identiteitsemoji van de agent (`agents.entries.*.identity.emoji`, anders "👀")

    Opmerkingen:

    - Discord accepteert Unicode-emoji of namen van aangepaste emoji.
    - Gebruik `""` om de reactie voor een kanaal of account uit te schakelen.

    **Bereik (`messages.ackReactionScope`):**

    Waarden: `"all"` (DM's + groepen, inclusief omgevingsgebeurtenissen in ruimtes), `"direct"` (alleen DM's), `"group-all"` (elk groepsbericht behalve omgevingsgebeurtenissen in ruimtes, geen DM's), `"group-mentions"` (groepen wanneer de bot wordt vermeld; **geen DM's**, standaard), `"off"` / `"none"` (uitgeschakeld).

    <Note>
    Het standaardbereik (`"group-mentions"`) activeert geen bevestigingsreacties in directe berichten of bij omgevingsgebeurtenissen in ruimtes. Stel `messages.ackReactionScope` in op `"all"` om een bevestigingsreactie te krijgen bij inkomende Discord-DM's en stille ruimtegebeurtenissen.
    </Note>

  </Accordion>

  <Accordion title="Configuratie schrijven">
    Door het kanaal geïnitieerde configuratiewijzigingen zijn standaard ingeschakeld. Dit beïnvloedt `/config set|unset`-flows (wanneer opdrachtfuncties zijn ingeschakeld).

    Uitschakelen:

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Gateway-proxy">
    Routeer Discord Gateway-WebSocket-verkeer en REST-opzoekacties bij het opstarten (toepassings-ID + bepaling van de toelatingslijst) via een HTTP(S)-proxy met `channels.discord.proxy`.
    Proxygebruik voor Discord Gateway-WebSockets is expliciet; WebSocket-verbindingen nemen geen proxyomgevingsvariabelen over uit het Gateway-proces. REST-opzoekacties bij het opstarten gebruiken deze proxy wanneer `channels.discord.proxy` is geconfigureerd.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    Overschrijving per account:

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit-ondersteuning">
    Schakel PluralKit-resolutie in om proxyberichten aan de identiteit van een systeemlid te koppelen:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // optional; needed for private systems
      },
    },
  },
}
```

    Opmerkingen:

    - toelatingslijsten kunnen `pk:<memberId>` gebruiken
    - weergavenamen van leden worden alleen op naam/slug vergeleken wanneer `channels.discord.dangerouslyAllowNameMatching: true`
    - opzoekacties raadplegen de PluralKit-API met het oorspronkelijke bericht-ID
    - als de opzoekactie mislukt, worden proxyberichten als botberichten behandeld en verwijderd, tenzij `allowBots` ze doorlaat

  </Accordion>

  <Accordion title="Aliassen voor uitgaande vermeldingen">
    Gebruik `mentionAliases` wanneer agents deterministische uitgaande vermeldingen voor bekende Discord-gebruikers nodig hebben. Sleutels zijn handles zonder de voorafgaande `@`; waarden zijn Discord-gebruikers-ID's. Onbekende handles, `@everyone`, `@here` en vermeldingen binnen Markdown-codefragmenten blijven ongewijzigd.

```json5
{
  channels: {
    discord: {
      mentionAliases: {
        SupportLead: "123456789012345678",
      },
      accounts: {
        ops: {
          mentionAliases: {
            OpsLead: "234567890123456789",
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="Aanwezigheidsconfiguratie">
    Aanwezigheidsupdates worden toegepast wanneer je een status- of activiteitsveld instelt, of wanneer je automatische aanwezigheid inschakelt.

    Alleen status:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    Activiteit (aangepaste status is het standaardactiviteitstype wanneer `activity` is ingesteld):

```json5
{
  channels: {
    discord: {
      activity: "Focus time",
      activityType: 4,
    },
  },
}
```

    Streamen:

```json5
{
  channels: {
    discord: {
      activity: "Live coding",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    Overzicht van activiteitstypen:

    - 0: Spelen
    - 1: Streamen (vereist `activityUrl`; `activityUrl` vereist op zijn beurt `activityType: 1`)
    - 2: Luisteren
    - 3: Kijken
    - 4: Aangepast (gebruikt de activiteitstekst als statustekst; emoji is optioneel)
    - 5: Deelnemen aan een wedstrijd

    Automatische aanwezigheid (statussignaal van de runtime):

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token exhausted",
      },
    },
  },
}
```

    Automatische aanwezigheid koppelt de beschikbaarheid van de runtime aan de Discord-status: gezond => online, verslechterd of onbekend => inactief, uitgeput of niet beschikbaar => niet storen. Standaardwaarden: `intervalMs` 30000, `minUpdateIntervalMs` 15000 (moet kleiner dan of gelijk aan `intervalMs` zijn). Optionele tekstoverschrijvingen:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (ondersteunt de tijdelijke aanduiding `{reason}`)

  </Accordion>

  <Accordion title="Goedkeuringen in Discord">
    Discord ondersteunt afhandeling van goedkeuringen via knoppen in DM's en kan goedkeuringsverzoeken optioneel in het oorspronkelijke kanaal plaatsen.

    Configuratiepad:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (optioneel; valt waar mogelijk terug op `commands.ownerAllowFrom`)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`, standaard: `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    Discord schakelt native uitvoeringsgoedkeuringen automatisch in wanneer `enabled` niet is ingesteld of `"auto"` is en ten minste één goedkeurder kan worden bepaald, hetzij via `execApprovals.approvers`, hetzij via `commands.ownerAllowFrom`. Discord leidt uitvoeringsgoedkeurders niet af uit kanaal-`allowFrom`, verouderde `dm.allowFrom` of directe-bericht-`defaultTo`. Stel `enabled: false` in om Discord expliciet uit te schakelen als native goedkeuringsclient.

    Voor gevoelige groepsopdrachten die uitsluitend voor eigenaren bestemd zijn, zoals `/diagnostics` en `/export-trajectory`, verzendt OpenClaw goedkeuringsverzoeken en eindresultaten privé. Eerst wordt een Discord-DM geprobeerd wanneer de eigenaar die de opdracht aanroept een Discord-eigenaarsroute heeft; anders wordt teruggevallen op de eerste beschikbare eigenaarsroute uit `commands.ownerAllowFrom`, zoals Telegram.

    Wanneer `target` `channel` of `both` is, is het goedkeuringsverzoek zichtbaar in het kanaal. Alleen bepaalde goedkeurders kunnen de knoppen gebruiken; andere gebruikers ontvangen een tijdelijke afwijzing die alleen voor hen zichtbaar is. Goedkeuringsverzoeken bevatten de opdrachttekst, dus schakel bezorging in het kanaal alleen in voor vertrouwde kanalen. Als het kanaal-ID niet uit de sessiesleutel kan worden afgeleid, valt OpenClaw terug op bezorging via DM.

    Discord geeft de gedeelde goedkeuringsknoppen weer die ook door andere chatkanalen worden gebruikt; de native Discord-adapter voegt hoofdzakelijk DM-routering voor goedkeurders en verspreiding naar kanalen toe. Wanneer die knoppen aanwezig zijn, vormen ze de primaire gebruikerservaring voor goedkeuringen; OpenClaw mag alleen een handmatige `/approve`-opdracht opnemen wanneer het toolresultaat aangeeft dat chatgoedkeuringen niet beschikbaar zijn of handmatige goedkeuring de enige mogelijkheid is. Als de native Discord-runtime voor goedkeuringen niet actief is, houdt OpenClaw de lokale deterministische `/approve <id> <decision>`-prompt zichtbaar. Als de runtime actief is maar een native kaart niet bij een doel kan worden bezorgd, verzendt OpenClaw in dezelfde chat een terugvalmelding met de exacte `/approve`-opdracht uit de wachtende goedkeuring.

    Gateway-authenticatie en bepaling van goedkeuringen volgen het gedeelde contract van de Gateway-client (`plugin:`-ID's worden bepaald via `plugin.approval.resolve`; andere ID's via `exec.approval.resolve`). Goedkeuringen verlopen standaard na 30 minuten.

    Zie [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals).

  </Accordion>
</AccordionGroup>

## Tools en actiepoorten

Discord-berichtacties omvatten berichten, kanaalbeheer, moderatie, aanwezigheid en metagegevens.

Kernvoorbeelden:

- berichten: `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- reacties: `react`, `reactions`, `emojiList`
- moderatie: `timeout`, `kick`, `ban`
- aanwezigheid: `setPresence`

De actie `event-create` accepteert een optionele parameter `image` (URL of lokaal bestandspad) om de omslagafbeelding voor de geplande gebeurtenis in te stellen.

Actiepoorten bevinden zich onder `channels.discord.actions.*`.

Standaardgedrag van poorten:

| Actiegroep                                                                                                                                                               | Standaard     |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | ingeschakeld  |
| roles                                                                                                                                                                    | uitgeschakeld |
| moderation                                                                                                                                                               | uitgeschakeld |
| presence                                                                                                                                                                 | uitgeschakeld |

## UI met Components v2

OpenClaw gebruikt Discord Components v2 voor uitvoeringsgoedkeuringen en contextoverschrijdende markeringen. Discord-berichtacties kunnen ook `components` accepteren voor een aangepaste UI (geavanceerd; hiervoor moet via de discord-tool een componentpayload worden samengesteld), terwijl verouderde `embeds` beschikbaar blijven, maar niet worden aanbevolen.

- `channels.discord.ui.components.accentColor` stelt de accentkleur in die door Discord-componentcontainers wordt gebruikt (hex). Per account: `channels.discord.accounts.<id>.ui.components.accentColor`.
- `channels.discord.agentComponents.ttlMs` bepaalt hoelang callbacks van verzonden Discord-componenten geregistreerd blijven (standaard `1800000`, maximaal `86400000`). Per account: `channels.discord.accounts.<id>.agentComponents.ttlMs`.
- `embeds` worden genegeerd wanneer Components v2 aanwezig zijn.
- Voorvertoningen van gewone URL's worden standaard onderdrukt. Stel `suppressEmbeds: false` in voor een berichtactie wanneer één uitgaande link moet worden uitgevouwen.

Voorbeeld:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## Spraak

Discord heeft twee afzonderlijke spraakoppervlakken: realtime **spraakkanalen** (doorlopende gesprekken) en **spraakberichtbijlagen** (de indeling met golfvormvoorvertoning). De Gateway ondersteunt beide.

### Spraakkanalen

Controlelijst voor de configuratie:

1. Schakel Message Content Intent in de Discord Developer Portal in.
2. Schakel Server Members Intent in wanneer toelatingslijsten voor rollen/gebruikers worden gebruikt.
3. Nodig de bot uit met de scopes `bot` en `applications.commands`.
4. Verleen Connect, Speak, Send Messages en Read Message History in het doelspraakkanaal.
5. Schakel native opdrachten in (`commands.native` of `channels.discord.commands.native`).
6. Configureer `channels.discord.voice`.

Gebruik `/vc join|leave|status` om sessies te beheren. De opdracht gebruikt de standaardagent van het account en volgt dezelfde regels voor toelatingslijsten en groepsbeleid als andere Discord-opdrachten.

```bash
/vc join channel:<voice-channel-id>
/vc status
/vc leave
```

Om de effectieve machtigingen van de bot te controleren voordat deze deelneemt:

```bash
openclaw channels capabilities --channel discord --target channel:<voice-channel-id>
```

Voorbeeld van automatisch deelnemen:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

Opmerkingen:

- Discord-spraak is opt-in voor configuraties met alleen tekst; stel `channels.discord.voice.enabled=true` in (of behoud een bestaand `channels.discord.voice`-blok) om `/vc`-opdrachten, de spraakruntime en de `GuildVoiceStates`-gatewayintentie in te schakelen. `channels.discord.intents.voiceStates` kan het intentieabonnement expliciet overschrijven; laat dit oningesteld om de effectieve inschakeling van spraak te volgen.
- `voice.mode` bepaalt het gesprekspad. De standaard is `agent-proxy`: een realtime spraakfrontend verwerkt de timing van gespreksbeurten, onderbrekingen en het afspelen, delegeert inhoudelijk werk via `openclaw_agent_consult` aan de gerouteerde OpenClaw-agent en behandelt het resultaat alsof het een getypte Discord-prompt van die spreker is. `stt-tts` behoudt de oudere batchflow met STT plus TTS. Met `bidi` kan het realtimemodel rechtstreeks converseren, terwijl `openclaw_agent_consult` beschikbaar wordt gesteld voor het OpenClaw-brein.
- `voice.agentSession` bepaalt welk OpenClaw-gesprek spraakbeurten ontvangt. Laat dit oningesteld voor de eigen sessie van het spraakkanaal, of stel `{ mode: "target", target: "channel:<text-channel-id>" }` in om het spraakkanaal te laten fungeren als microfoon-/luidsprekeruitbreiding van een bestaande Discord-tekstkanaalsessie, zoals `#maintainers`.
- `voice.model` overschrijft het OpenClaw-agentbrein voor Discord-spraakantwoorden en realtime raadplegingen. Laat dit oningesteld om het gerouteerde agentmodel over te nemen. Dit staat los van `voice.realtime.model`.
- Met `voice.followUsers` kan de bot zich bij geselecteerde gebruikers aansluiten bij Discord-spraak, met hen meeverhuizen en het spraakkanaal verlaten. Zie [Gebruikers volgen in spraak](#follow-users-in-voice).
- `agent-proxy` routeert spraak via `discord-voice`, waarbij de normale autorisatie van de eigenaar en tools voor de spreker en doelsessie behouden blijft, maar de agenttool `tts` wordt verborgen omdat Discord-spraak het afspelen beheert. Standaard geeft `agent-proxy` de raadpleging volledige, aan de eigenaar gelijkwaardige tooltoegang voor sprekers die eigenaar zijn (`voice.realtime.toolPolicy: "owner"`) en wordt sterk de voorkeur gegeven aan het raadplegen van de OpenClaw-agent vóór inhoudelijke antwoorden (`voice.realtime.consultPolicy: "always"`). In die standaardmodus `always` spreekt de realtimelaag niet automatisch opvultekst uit vóór het antwoord van de raadpleging; de laag legt spraak vast en transcribeert deze, en spreekt vervolgens het gerouteerde OpenClaw-antwoord uit. Als meerdere afgedwongen raadpleegantwoorden worden voltooid terwijl Discord het eerste antwoord nog afspeelt, worden latere antwoorden met exacte spraak in de wachtrij geplaatst totdat het afspelen inactief is, in plaats van spraak halverwege een zin te vervangen.
- In de modus `stt-tts` gebruikt STT `tools.media.audio`; `voice.model` heeft geen invloed op de transcriptie.
- In realtimemodi configureren `voice.realtime.provider`, `voice.realtime.model` en `voice.realtime.speakerVoice` de realtime audiosessie. Gebruik voor OpenAI Realtime 2.1 plus het Codex-brein `voice.realtime.model: "gpt-realtime-2.1"` en `voice.model: "openai/gpt-5.6-sol"`.
- Realtimespraakmodi nemen standaard kleine profielbestanden `IDENTITY.md`, `USER.md` en `SOUL.md` op in de instructies voor de realtimeprovider, zodat snelle rechtstreekse beurten dezelfde identiteit, gebruikerscontext en persona behouden als de gerouteerde OpenClaw-agent. Stel `voice.realtime.bootstrapContextFiles` in op een subset om dit aan te passen, of `[]` om dit uit te schakelen. Alleen die profielbestanden worden ondersteund; `AGENTS.md` blijft in de normale agentcontext. De geïnjecteerde profielcontext vervangt `openclaw_agent_consult` niet voor werk in de werkruimte, actuele feiten, het opzoeken van geheugen of door tools ondersteunde acties.
- In de realtime OpenAI-modus `agent-proxy` past de activering op weknaam zich standaard aan de ruimte aan: één persoon kan natuurlijk spreken zonder weknaam, terwijl twee of meer personen een beurt met een weknaam moeten beginnen of eindigen. Andere bots tellen niet als personen. Stel `voice.realtime.requireWakeName: true` in om altijd een weknaam te vereisen of `false` om er nooit een te vereisen. Geconfigureerde weknamen moeten uit één of twee woorden bestaan. Als `voice.realtime.wakeNames` niet is ingesteld, gebruikt OpenClaw de `name` van de gerouteerde agent plus `OpenClaw`, met als terugval de agent-id plus `OpenClaw`. Een actieve weknaamcontrole schakelt automatische antwoorden van de realtimeprovider uit, routeert geaccepteerde beurten via het raadpleegpad van de OpenClaw-agent en geeft een korte gesproken bevestiging wanneer een weknaam aan het begin wordt herkend uit een gedeeltelijke transcriptie voordat het definitieve transcript binnenkomt. Het beleid volgt live deelnemers die binnenkomen en vertrekken zonder opnieuw verbinding met spraak te maken.
- De realtimeprovider van OpenAI accepteert huidige Realtime 2-gebeurtenisnamen en verouderde Codex-compatibele aliassen voor uitvoeraudio- en transcriptgebeurtenissen, zodat compatibele providersnapshots kunnen afwijken zonder assistentaudio weg te laten.
- `voice.realtime.bargeIn` bepaalt of Discord-gebeurtenissen voor het begin van spreken het actieve realtime afspelen onderbreken. Als dit niet is ingesteld, volgt het de instelling van de realtimeprovider voor onderbreking door invoeraudio.
- `voice.realtime.minBargeInAudioEndMs` bepaalt de minimale afspeelduur van de assistent voordat een onderbreking in OpenAI-realtime de audio afkapt. Standaard: `250`. Stel `0` in voor onmiddellijke onderbreking in ruimtes met weinig echo, of verhoog dit voor luidsprekeropstellingen met veel echo.
- `voice.tts` overschrijft `tts` alleen voor het afspelen van `stt-tts`-spraak; realtimemodi gebruiken in plaats daarvan `voice.realtime.speakerVoice`. Stel voor een OpenAI-stem bij afspelen via Discord `voice.tts.provider: "openai"` in en kies een Text-to-speech-stem onder `voice.tts.providers.openai.speakerVoice`. `cedar` is een goede mannelijk klinkende keuze voor het huidige OpenAI TTS-model.
- Discord-overschrijvingen per kanaal in `systemPrompt` zijn van toepassing op spraaktranscriptiebeurten voor dat spraakkanaal.
- Wanneer OpenClaw zich bij een spraakkanaal aansluit, ontvangt de gerouteerde agentsessie een stille systeemgebeurtenis met de huidige deelnemerslijst. Latere deelnemers die binnenkomen en vertrekken werken die sessie bij zonder een ongevraagd gesproken antwoord te activeren; Discord-weergavenamen worden behandeld als niet-vertrouwde labels. Geautoriseerde spraakbeurten ontvangen ook een actuele momentopname van de deelnemerslijst.
- Spraaktranscriptiebeurten en `/vc`-opdrachten gebruiken Discord-vermeldingen in `commands.ownerAllowFrom` voor de eigenaarsstatus. Wanneer er geen eigenaar voor Discord-opdrachten is geconfigureerd, kan de `allowFrom` (of verouderde `dm.allowFrom`) van het geselecteerde Discord-account nog steeds spraaktoegang autoriseren zonder eigenaarsstatus toe te kennen. De zichtbaarheid van agenttools volgt het geconfigureerde toolbeleid voor de gerouteerde sessie.
- Als `voice.autoJoin` meerdere vermeldingen voor dezelfde guild bevat, sluit OpenClaw zich aan bij het laatst geconfigureerde kanaal voor die guild.
- `voice.allowedChannels` is een optionele toelatingslijst voor verblijf. Laat dit oningesteld om `/vc join` in elk geautoriseerd Discord-spraakkanaal toe te staan. Wanneer dit is ingesteld, worden `/vc join`, automatisch aansluiten bij het opstarten en verplaatsingen van de spraakstatus van de bot beperkt tot de vermelde `{ guildId, channelId }`-vermeldingen. Stel dit in op een lege array om alle aansluitingen bij Discord-spraak te weigeren. Als Discord de bot buiten de toelatingslijst verplaatst, verlaat OpenClaw dat kanaal en sluit het zich opnieuw aan bij het geconfigureerde doel voor automatisch aansluiten wanneer er een beschikbaar is.
- `voice.daveEncryption` en `voice.decryptionFailureTolerance` worden doorgegeven aan de aansluitopties van `@discordjs/voice`; de bovenliggende standaardwaarden zijn `daveEncryption=true` en `decryptionFailureTolerance=24`.
- OpenClaw gebruikt de meegeleverde `libopus-wasm`-codec voor de ontvangst van Discord-spraak en het realtime afspelen van onbewerkte PCM. Deze wordt geleverd met een vastgezette libopus-WebAssembly-build en vereist geen native opus-add-ons.
- `voice.connectTimeoutMs` bepaalt de initiële wachttijd op `@discordjs/voice` Ready voor `/vc join` en pogingen om automatisch aan te sluiten. Standaard: `30000`.
- `voice.reconnectGraceMs` bepaalt hoelang OpenClaw wacht totdat een verbroken spraaksessie opnieuw verbinding begint te maken voordat de sessie wordt vernietigd. Standaard: `15000`.
- In de modus `stt-tts` stopt het afspelen van spraak niet alleen omdat een andere gebruiker begint te spreken. Om feedbacklussen te voorkomen, negeert OpenClaw nieuwe spraakopname terwijl TTS wordt afgespeeld; spreek nadat het afspelen is voltooid voor de volgende beurt. Realtimemodi sturen het begin van spreken door als onderbrekingssignalen naar de realtimeprovider.
- In realtimemodi kan echo van luidsprekers naar een open microfoon op een onderbreking lijken en het afspelen onderbreken. Stel voor Discord-ruimtes met veel echo `voice.realtime.providers.openai.interruptResponseOnInputAudio: false` in om te voorkomen dat OpenAI automatisch onderbreekt bij invoeraudio. Voeg `voice.realtime.bargeIn: true` toe als je nog steeds wilt dat Discord-gebeurtenissen voor het begin van spreken het actieve afspelen onderbreken. De OpenAI-realtimebridge negeert afkappingen van het afspelen die korter zijn dan `voice.realtime.minBargeInAudioEndMs` als waarschijnlijke echo/ruis en registreert ze als overgeslagen in plaats van het afspelen via Discord te wissen.
- `voice.captureSilenceGraceMs` bepaalt hoelang OpenClaw wacht nadat Discord meldt dat een spreker is gestopt voordat dat audiosegment voor STT wordt afgerond. Standaard: `2000`; verhoog dit als Discord normale pauzes opsplitst in hakkelende gedeeltelijke transcripties.
- Wanneer ElevenLabs de geselecteerde TTS-provider is, gebruikt het afspelen van Discord-spraak streaming-TTS en begint het vanuit de antwoordstream van de provider. Providers zonder ondersteuning voor streaming vallen terug op het pad met een gesynthetiseerd tijdelijk bestand.
- OpenClaw bewaakt fouten bij het ontsleutelen van ontvangen gegevens en herstelt automatisch door het spraakkanaal te verlaten en er opnieuw aan deel te nemen na herhaalde fouten binnen een kort tijdsvenster.
- Als ontvangstlogboeken na een update herhaaldelijk `DecryptionFailed(UnencryptedWhenPassthroughDisabled)` tonen, verzamel dan een afhankelijkheidsrapport en logboeken. De meegeleverde `@discordjs/voice`-regel bevat de bovenliggende opvullingscorrectie uit discord.js-PR #11449, waarmee discord.js-issue #11419 werd gesloten.
- `The operation was aborted`-ontvangstgebeurtenissen worden verwacht wanneer OpenClaw een vastgelegd sprekersegment afrondt; het zijn uitgebreide diagnostische meldingen, geen waarschuwingen.
- Uitgebreide Discord-spraaklogboeken bevatten voor elk geaccepteerd sprekersegment een begrensd STT-transcriptievoorbeeld van één regel, zodat bij foutopsporing zowel de gebruikerszijde als de antwoordzijde van de agent zichtbaar is zonder onbeperkte transcriptietekst te dumpen.
- In de modus `agent-proxy` slaat de terugval voor afgedwongen raadpleging waarschijnlijk onvolledige transcriptiefragmenten over, zoals tekst die eindigt op `...` of een afsluitend verbindingswoord zoals "en", evenals duidelijk niet-uitvoerbare afsluitingen zoals "ben zo terug" of "dag". Logboeken tonen `forced agent consult skipped reason=...` wanneer dit een verouderd antwoord in de wachtrij voorkomt.

### Gebruikers volgen in spraak

Gebruik `voice.followUsers` wanneer je wilt dat de Discord-spraakbot bij een of meer bekende Discord-gebruikers blijft, in plaats van zich bij het opstarten bij een vast kanaal aan te sluiten of op `/vc join` te wachten.

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        followUsersEnabled: true,
        followUsers: ["discord:123456789012345678"],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
      },
    },
  },
}
```

Gedrag:

- `followUsers` accepteert onbewerkte Discord-gebruikers-ID's en `discord:<id>`-waarden. OpenClaw normaliseert beide vormen voordat spraakstatusgebeurtenissen worden vergeleken.
- `followUsersEnabled` wordt standaard ingesteld op `true` wanneer `followUsers` is geconfigureerd. Stel dit in op `false` om de opgeslagen lijst te behouden, maar het automatisch volgen van spraak te stoppen.
- `followUsers` regelt alleen de aanwezigheid in spraakkanalen. Het verleent geen toegang als spreker of eigenaarsbevoegdheid; configureer `commands.ownerAllowFrom` en gebruikers en rollen voor de server of het kanaal afzonderlijk.
- Wanneer een gevolgde gebruiker deelneemt aan een toegestaan spraakkanaal, neemt OpenClaw deel aan dat kanaal. Wanneer de gebruiker zich verplaatst, verplaatst OpenClaw zich mee. Wanneer de actief gevolgde gebruiker de verbinding verbreekt, verlaat OpenClaw het kanaal.
- Als meerdere gevolgde gebruikers zich op dezelfde server bevinden en de actief gevolgde gebruiker vertrekt, verplaatst OpenClaw zich naar het kanaal van een andere bijgehouden gevolgde gebruiker voordat de server wordt verlaten. Als meerdere gevolgde gebruikers zich tegelijk verplaatsen, is de laatst waargenomen spraakstatusgebeurtenis bepalend.
- `allowedChannels` blijft van toepassing. Een gevolgde gebruiker in een niet-toegestaan kanaal wordt genegeerd en een sessie die door volgen wordt beheerd, verplaatst zich naar een andere gevolgde gebruiker of vertrekt.
- OpenClaw reconcilieert gemiste spraakstatusgebeurtenissen bij het opstarten en met een begrensd interval. Bij reconciliatie worden geconfigureerde servers steekproefsgewijs gecontroleerd en wordt het aantal REST-opzoekacties per uitvoering begrensd. Daardoor kan het bij zeer grote `followUsers`-lijsten meer dan één interval duren voordat de status overeenkomt.
- Als Discord of een beheerder de bot verplaatst terwijl deze een gebruiker volgt, bouwt OpenClaw de spraaksessie opnieuw op en blijft het volgen de sessie beheren als de bestemming is toegestaan. Als de bot buiten `allowedChannels` wordt verplaatst, vertrekt OpenClaw en neemt het opnieuw deel aan het geconfigureerde doel wanneer dat bestaat.
- Bij DAVE-ontvangstherstel kan hetzelfde kanaal na herhaalde ontsleutelingsfouten worden verlaten en opnieuw worden betreden. Sessies die door volgen worden beheerd, blijven tijdens dat herstelpad door volgen beheerd, zodat het kanaal alsnog wordt verlaten wanneer een gevolgde gebruiker later de verbinding verbreekt.

Kies uit de deelnamemodi:

- Gebruik `followUsers` voor persoonlijke configuraties of operatorconfiguraties waarbij de bot automatisch in het spraakkanaal aanwezig moet zijn wanneer jij dat bent.
- Gebruik `autoJoin` voor bots in een vaste ruimte die aanwezig moeten zijn, zelfs wanneer geen bijgehouden gebruiker in een spraakkanaal aanwezig is.
- Gebruik `/vc join` voor eenmalige deelnames of ruimtes waar automatische aanwezigheid in een spraakkanaal onverwacht zou zijn.

Discord-spraakcodec:

- Logboeken voor spraakontvangst tonen `discord voice: opus decoder: libopus-wasm`.
- Realtime afspelen codeert onbewerkte 48 kHz stereo-PCM naar Opus met hetzelfde meegeleverde `libopus-wasm`-pakket voordat pakketten aan `@discordjs/voice` worden doorgegeven.
- Bij het afspelen van bestanden en providerstreams wordt met ffmpeg getranscodeerd naar onbewerkte 48 kHz stereo-PCM. Vervolgens wordt `libopus-wasm` gebruikt voor de Opus-pakketstream die naar Discord wordt verzonden.

STT- plus TTS-pijplijn:

- De Discord-PCM-opname wordt omgezet in een tijdelijk WAV-bestand.
- `tools.media.audio` verwerkt STT, bijvoorbeeld `openai/gpt-4o-mini-transcribe`.
- Het transcript wordt via de Discord-ingang en routering verzonden, terwijl het LLM voor het antwoord wordt uitgevoerd met een beleid voor spraakuitvoer dat het `tts`-hulpmiddel van de agent verbergt en om geretourneerde tekst vraagt, omdat Discord-spraak het uiteindelijke TTS-afspelen beheert.
- `voice.model` overschrijft, wanneer ingesteld, alleen het LLM voor het antwoord tijdens deze beurt in het spraakkanaal.
- `voice.tts` wordt over `tts` samengevoegd; providers die streaming ondersteunen, leveren rechtstreeks aan de speler. Anders wordt het resulterende audiobestand in het kanaal afgespeeld waaraan is deelgenomen.

Voorbeeld van een standaard spraakkanaalsessie met agentproxy:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        followUsersEnabled: true,
        followUsers: ["123456789012345678"],
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

Zonder `voice.agentSession`-blok krijgt elk spraakkanaal een eigen gerouteerde OpenClaw-sessie. `/vc join channel:234567890123456789` communiceert bijvoorbeeld met de sessie voor dat Discord-spraakkanaal. Het realtime model is alleen de spraakfrontend; inhoudelijke verzoeken worden doorgegeven aan de geconfigureerde OpenClaw-agent. Als het realtime model een definitief transcript produceert zonder het consultatiehulpmiddel aan te roepen, dwingt OpenClaw als terugval de consultatie af, zodat de standaardwerking nog steeds overeenkomt met praten met de agent.

Voorbeeld van verouderde STT plus TTS:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "stt-tts",
        model: "openai/gpt-5.4-mini",
        tts: {
          provider: "openai",
          providers: {
            openai: {
              model: "gpt-4o-mini-tts",
              speakerVoice: "cedar",
            },
          },
        },
      },
    },
  },
}
```

Voorbeeld van bidirectionele realtimecommunicatie:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          toolPolicy: "safe-read-only",
          consultPolicy: "always",
        },
      },
    },
  },
}
```

Spraak als uitbreiding van een bestaande Discord-kanaalsessie:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "agent-proxy",
        model: "openai/gpt-5.6-sol",
        agentSession: {
          mode: "target",
          target: "channel:123456789012345678",
        },
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

In de modus `agent-proxy` neemt de bot deel aan het geconfigureerde spraakkanaal, maar gebruiken OpenClaw-agentbeurten de normale gerouteerde sessie en agent van het doelkanaal. De realtime spraaksessie spreekt het geretourneerde resultaat weer uit in het spraakkanaal. De supervisoragent kan volgens het eigen hulpmiddelenbeleid nog steeds normale berichthulpmiddelen gebruiken, waaronder het verzenden van een afzonderlijk Discord-bericht als dat de juiste actie is.

Terwijl een gedelegeerde OpenClaw-uitvoering actief is, worden nieuwe Discord-spraaktranscripten verwerkt als live besturing van de uitvoering voordat een nieuwe agentbeurt wordt gestart. Zinnen zoals "status", "annuleer dat", "gebruik de kleinere oplossing" of "controleer ook de tests wanneer je klaar bent" worden geclassificeerd als status-, annulerings-, bijsturings- of vervolginvoer voor de actieve sessie. Resultaten van status, annulering, geaccepteerde bijsturing en vervolgacties worden uitgesproken in het spraakkanaal, zodat de beller weet of OpenClaw het verzoek heeft verwerkt.

Nuttige doelvormen:

- `target: "channel:123456789012345678"` routeert via een Discord-tekstkanaalsessie.
- `target: "123456789012345678"` wordt als kanaaldoel behandeld.
- `target: "dm:123456789012345678"` of `target: "user:123456789012345678"` routeert via die privéberichtsessie.

Voorbeeld van OpenAI Realtime met veel echo:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          bargeIn: true,
          minBargeInAudioEndMs: 500,
          consultPolicy: "always",
          providers: {
            openai: {
              interruptResponseOnInputAudio: false,
            },
          },
        },
      },
    },
  },
}
```

Gebruik dit wanneer het model zijn eigen Discord-weergave via een open microfoon hoort, maar je het nog steeds wilt kunnen onderbreken door te spreken. OpenClaw voorkomt dat OpenAI automatisch onderbreekt bij onbewerkte audio-invoer, terwijl `bargeIn: true` ervoor zorgt dat Discord-gebeurtenissen voor het begin van een spreker en reeds actieve sprekeraudio actieve realtime antwoorden kunnen annuleren voordat de volgende opgenomen beurt OpenAI bereikt. Zeer vroege onderbrekingssignalen waarbij `audioEndMs` lager is dan `minBargeInAudioEndMs`, worden als waarschijnlijke echo of ruis beschouwd en genegeerd, zodat het model niet al bij het eerste afspeelframe wordt afgebroken.

Verwachte spraaklogboeken:

- Bij deelname: `discord voice: joining ... voiceSession=... supervisorSession=... agentSessionMode=... voiceModel=... realtimeModel=...`
- Bij het starten van realtime: `discord voice: realtime bridge starting ... autoRespond=false interruptResponse=false bargeIn=false minBargeInAudioEndMs=...`
- Bij sprekeraudio: `discord voice: realtime speaker turn opened ...`, `discord voice: realtime input audio started ... outputAudioMs=... outputActive=...` en `discord voice: realtime speaker turn closed ... chunks=... discordBytes=... realtimeBytes=... interruptedPlayback=...`
- Bij overgeslagen verouderde spraak: `discord voice: realtime forced agent consult skipped reason=incomplete-transcript ...` of `reason=non-actionable-closing ...`
- Bij voltooiing van een realtime antwoord: `discord voice: realtime audio playback finishing reason=response.done ... audioMs=... chunks=...`
- Bij stoppen/herstellen van het afspelen: `discord voice: realtime audio playback stopped reason=... audioMs=... elapsedMs=... chunks=...`
- Bij realtime consultatie: `discord voice: realtime consult requested ... voiceSession=... supervisorSession=... question=...`
- Bij antwoord van de agent: `discord voice: agent turn answer ...`
- Bij exact in de wachtrij geplaatste spraak: `discord voice: realtime exact speech queued ... queued=... outputAudioMs=... outputActive=...`, gevolgd door `discord voice: realtime exact speech dequeued reason=player-idle ...`
- Bij detectie van een onderbreking: `discord voice: realtime barge-in detected source=speaker-start ...` of `discord voice: realtime barge-in detected source=active-speaker-audio ...`, gevolgd door `discord voice: realtime barge-in requested reason=... outputAudioMs=... outputActive=...`
- Bij realtime onderbreking: `discord voice: realtime model interrupt requested client:response.cancel reason=barge-in`, gevolgd door `discord voice: realtime model audio truncated client:conversation.item.truncate reason=barge-in audioEndMs=...` of `discord voice: realtime model interrupt confirmed server:response.done status=cancelled ...`
- Bij genegeerde echo/ruis: `discord voice: realtime model interrupt ignored client:conversation.item.truncate.skipped reason=barge-in audioEndMs=0 minAudioEndMs=250`
- Bij uitgeschakelde onderbreking: `discord voice: realtime capture ignored during playback (barge-in disabled) ...`
- Bij inactief afspelen: `discord voice: realtime barge-in ignored reason=... outputActive=false ... playbackChunks=0`

Lees voor het opsporen van voortijdig afgebroken audio de realtime spraaklogboeken als een tijdlijn:

1. `realtime audio playback started` betekent dat Discord is begonnen met het afspelen van assistentaudio. Vanaf dit punt begint de bridge assistentuitvoerblokken, Discord-PCM-bytes, realtime providerbytes en de duur van gesynthetiseerde audio te tellen.
2. `realtime speaker turn opened` markeert dat een Discord-spreker actief wordt. Als het afspelen al actief is en `bargeIn` is ingeschakeld, kan dit worden gevolgd door `barge-in detected source=speaker-start`.
3. `realtime input audio started` markeert het eerste daadwerkelijk ontvangen audioframe voor die sprekerbeurt. `outputActive=true` of een niet-nulwaarde voor `outputAudioMs` betekent hier dat de microfoon invoer verzendt terwijl het afspelen van de assistent nog actief is.
4. `barge-in detected source=active-speaker-audio` betekent dat OpenClaw live sprekeraudio heeft waargenomen terwijl het afspelen van de assistent actief was. Dit is nuttig om een echte onderbreking te onderscheiden van een Discord-gebeurtenis voor het begin van een spreker zonder bruikbare audio.
5. `barge-in requested reason=...` betekent dat OpenClaw de realtime provider heeft gevraagd het actieve antwoord te annuleren of af te kappen. Het bevat `outputAudioMs`, `outputActive` en `playbackChunks`, zodat je kunt zien hoeveel assistentaudio daadwerkelijk was afgespeeld vóór de onderbreking.
6. `realtime audio playback stopped reason=...` is het lokale herstelpunt voor het afspelen in Discord. De reden geeft aan wie het afspelen heeft gestopt: `barge-in`, `player-idle`, `provider-clear-audio`, `forced-agent-consult`, `stream-close` of `session-close`.
7. `realtime speaker turn closed` vat de opgenomen invoerbeurt samen. `chunks=0` of `hasAudio=false` betekent dat de sprekerbeurt is geopend, maar dat geen bruikbare audio de realtime bridge heeft bereikt. `interruptedPlayback=true` betekent dat die invoerbeurt de uitvoer van de assistent overlapte en de onderbrekingslogica activeerde.

Nuttige velden:

- `outputAudioMs`: duur van de assistentaudio die vóór de logregel door de realtime provider is gegenereerd.
- `audioMs`: duur van de assistentaudio die OpenClaw heeft geteld voordat het afspelen stopte.
- `elapsedMs`: verstreken kloktijd tussen het openen en sluiten van de afspeelstream of sprekerbeurt.
- `discordBytes`: 48 kHz stereo-PCM-bytes die naar Discord-spraak zijn verzonden of daarvan zijn ontvangen.
- `realtimeBytes`: PCM-bytes in providerindeling die naar de realtime provider zijn verzonden of daarvan zijn ontvangen.
- `playbackChunks`: assistentaudioblokken die voor het actieve antwoord naar Discord zijn doorgestuurd.
- `sinceLastAudioMs`: tijdsverschil tussen het laatste opgenomen sprekeraudioframe en het sluiten van de sprekerbeurt.

Veelvoorkomende patronen:

- Onmiddellijk afbreken met `source=active-speaker-audio`, een kleine `outputAudioMs` en dezelfde gebruiker in de buurt wijst meestal op luidsprekerecho die de microfoon binnenkomt. Verhoog `voice.realtime.minBargeInAudioEndMs`, verlaag het luidsprekervolume, gebruik een hoofdtelefoon of stel `voice.realtime.providers.openai.interruptResponseOnInputAudio: false` in.
- `source=speaker-start` gevolgd door `speaker turn closed ... hasAudio=false` betekent dat Discord meldde dat een spreker begon, maar dat OpenClaw geen audio ontving. Dit kan een tijdelijke Discord-spraakgebeurtenis zijn, gedrag van de ruispoort of een client die de microfoon kort activeert.
- `audio playback stopped reason=stream-close` zonder een nabijgelegen onderbreking of `provider-clear-audio` betekent dat de lokale Discord-afspeelstream onverwacht is beëindigd. Controleer de voorafgaande logboeken van de provider en de Discord-speler.
- `capture ignored during playback (barge-in disabled)` betekent dat OpenClaw invoer opzettelijk heeft genegeerd terwijl assistentaudio actief was. Schakel `voice.realtime.bargeIn` in als je wilt dat spraak het afspelen onderbreekt.
- `barge-in ignored ... outputActive=false` betekent dat Discord of de VAD van de provider spraak heeft gemeld, maar OpenClaw geen actieve afspeelsessie had om te onderbreken. Hierdoor mag audio niet worden afgebroken.

Aanmeldgegevens worden per component bepaald: LLM-routeauthenticatie voor `voice.model`, STT-authenticatie voor `tools.media.audio`, TTS-authenticatie voor `tts`/`voice.tts` en realtime providerauthenticatie voor `voice.realtime.providers` of de normale authenticatieconfiguratie van de provider.

### Spraakberichten

Discord-spraakberichten tonen een golfvormvoorbeeld en vereisen OGG/Opus-audio. OpenClaw genereert de golfvorm automatisch, maar heeft `ffmpeg` en `ffprobe` op de Gateway-host nodig om de audio te inspecteren en te converteren.

- Geef een **lokaal bestandspad** op (URL's worden geweigerd).
- Laat tekstinhoud weg (Discord weigert tekst en een spraakbericht in dezelfde payload).
- Elke audio-indeling wordt geaccepteerd; OpenClaw converteert zo nodig naar OGG/Opus.

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## Problemen oplossen

<AccordionGroup>
  <Accordion title="Niet-toegestane intents gebruikt of bot ziet geen serverberichten">

    - schakel Message Content Intent in
    - schakel Server Members Intent in wanneer je afhankelijk bent van gebruikers-/ledenresolutie
    - start de Gateway opnieuw nadat je intents hebt gewijzigd

  </Accordion>

  <Accordion title="Serverberichten onverwacht geblokkeerd">

    - controleer `groupPolicy`
    - controleer de servertoestaanlijst onder `channels.discord.guilds`
    - als er een `channels`-toewijzing voor een server bestaat, zijn alleen de vermelde kanalen toegestaan
    - controleer het gedrag van `requireMention` en de vermeldingspatronen

    Nuttige controles:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Vermelding niet vereist, maar toch geblokkeerd">
    Veelvoorkomende oorzaken:

    - `groupPolicy="allowlist"` zonder overeenkomende server-/kanaaltoestaanlijst
    - `requireMention` op de verkeerde plaats geconfigureerd (moet onder `channels.discord.guilds` of een kanaalvermelding staan)
    - afzender geblokkeerd door de `users`-toestaanlijst van de server of het kanaal

  </Accordion>

  <Accordion title="Langdurige Discord-beurten of dubbele antwoorden">

    Typische logboekvermeldingen:

    - `Slow listener detected ...`
    - `stuck session: sessionKey=agent:...:discord:... state=processing ...`

    Discord past geen time-out in eigendom van het kanaal toe op agentbeurten in de wachtrij. Berichtlisteners dragen taken onmiddellijk over en Discord-uitvoeringen in de wachtrij behouden de volgorde per sessie totdat de levenscyclus van de sessie, tool of runtime is voltooid of het werk afbreekt.

  </Accordion>

  <Accordion title="Waarschuwingen over time-outs bij het opzoeken van Gateway-metadata">
    OpenClaw haalt vóór het verbinden Discord-`/gateway/bot`-metadata op. Bij tijdelijke fouten wordt teruggevallen op de standaard-Gateway-URL van Discord en wordt het aantal logboekvermeldingen beperkt.

    De standaardtime-out voor metadata is 30 seconden. `OPENCLAW_DISCORD_GATEWAY_INFO_TIMEOUT_MS` kan deze voor ongebruikelijke hostomgevingen overschrijven.

  </Accordion>

  <Accordion title="Herstarts door time-out van Gateway READY">
    OpenClaw wacht tijdens het opstarten en na runtimeherverbindingen op de Gateway-gebeurtenis `READY` van Discord. Configuraties met meerdere accounts en gespreid opstarten kunnen een langer READY-venster bij het opstarten nodig hebben dan de standaardwaarde.

    Bij het opstarten wordt 15 seconden gewacht en bij runtimeherverbindingen 30 seconden. `OPENCLAW_DISCORD_READY_TIMEOUT_MS` en `OPENCLAW_DISCORD_RUNTIME_READY_TIMEOUT_MS` blijven beschikbaar voor ongebruikelijke hostomgevingen.

  </Accordion>

  <Accordion title="Afwijkingen bij de controle van machtigingen">
    Machtigingscontroles voor `channels status --probe` werken alleen met numerieke kanaal-ID's.

    Als je slug-sleutels gebruikt, kan runtimekoppeling nog steeds werken, maar kan de probe de machtigingen niet volledig verifiëren.

  </Accordion>

  <Accordion title="Problemen met privéberichten en koppeling">

    - privéberichten uitgeschakeld: `channels.discord.dm.enabled=false`
    - beleid voor privéberichten uitgeschakeld: `channels.discord.dmPolicy="disabled"` (verouderd: `channels.discord.dm.policy`)
    - wacht op goedkeuring van de koppeling in de modus `pairing`

  </Accordion>

  <Accordion title="Lussen tussen bots">
    Berichten van bots worden standaard genegeerd.

    Als je `channels.discord.allowBots=true` instelt, gebruik dan strikte regels voor vermeldingen en toestaanlijsten om lusgedrag te voorkomen.
    Geef de voorkeur aan `channels.discord.allowBots="mentions"` om alleen botberichten te accepteren waarin de bot wordt vermeld.

    OpenClaw bevat ook gedeelde [bescherming tegen botlussen](/nl/channels/bot-loop-protection). Wanneer `allowBots` berichten van bots doorlaat naar de dispatch, wijst Discord de inkomende gebeurtenis toe aan `(account, channel, bot pair)`-feiten en onderdrukt de generieke paarbeveiliging het paar nadat het geconfigureerde gebeurtenissenbudget is overschreden. De beveiliging voorkomt onbeheersbare lussen tussen twee bots die voorheen door Discord-snelheidslimieten moesten worden gestopt; deze heeft geen invloed op implementaties met één bot of eenmalige botantwoorden die onder het budget blijven.

    Standaardinstellingen (actief wanneer `allowBots` is ingesteld):

    - `maxEventsPerWindow: 20` -- het botpaar kan binnen het schuivende venster 20 berichten uitwisselen
    - `windowSeconds: 60` -- lengte van het schuivende venster
    - `cooldownSeconds: 60` -- zodra het budget wordt overschreden, wordt elk volgend bericht tussen bots in beide richtingen gedurende één minuut genegeerd

    Configureer de gedeelde standaardwaarde eenmaal onder `channels.defaults.botLoopProtection` en overschrijf deze vervolgens voor Discord wanneer een legitieme workflow meer ruimte nodig heeft. De prioriteitsvolgorde is:

    - `channels.discord.accounts.<account>.botLoopProtection`
    - `channels.discord.botLoopProtection`
    - `channels.defaults.botLoopProtection`
    - ingebouwde standaardwaarden

    Discord gebruikt de generieke sleutels `maxEventsPerWindow`, `windowSeconds` en `cooldownSeconds`.

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
    discord: {
      // Optionele overschrijving voor heel Discord. Accountblokken overschrijven afzonderlijke
      // velden en nemen weggelaten velden van hier over.
      botLoopProtection: {
        maxEventsPerWindow: 4,
      },
      accounts: {
        alpha: {
          // Alpha luistert alleen naar andere bots wanneer zij Alpha vermelden.
          allowBots: "mentions",
        },
        bravo: {
          // Bravo luistert naar alle Discord-berichten van bots.
          allowBots: true,
          mentionAliases: {
            // Hiermee kan Bravo een Discord-vermelding van Alpha schrijven met de geconfigureerde gebruikers-ID.
            Alpha: "ALPHA_DISCORD_USER_ID",
          },
          botLoopProtection: {
            // Sta maximaal vijf berichten per minuut toe voordat het paar wordt onderdrukt.
            maxEventsPerWindow: 5,
            windowSeconds: 60,
            cooldownSeconds: 90,
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="Spraak-STT valt weg met DecryptionFailed(...)">

    - houd OpenClaw actueel (`openclaw update`), zodat de herstellogica voor de ontvangst van Discord-spraak aanwezig is
    - bevestig `channels.discord.voice.daveEncryption=true` (standaard)
    - begin met `channels.discord.voice.decryptionFailureTolerance=24` (standaard van upstream) en pas dit alleen aan als dat nodig is
    - controleer de logboeken op:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - als de fouten na automatisch opnieuw deelnemen aanhouden, verzamel dan logboeken en vergelijk ze met de upstream DAVE-ontvangstgeschiedenis in [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) en [discord.js #11449](https://github.com/discordjs/discord.js/pull/11449)

  </Accordion>
</AccordionGroup>

## Configuratiereferentie

Primaire referentie: [Configuratiereferentie - Discord](/nl/gateway/config-channels#discord).

<Accordion title="Belangrijkste Discord-velden">

- opstarten/authenticatie: `enabled`, `token`, `applicationId`, `accounts.*`, `allowBots`
- beleid: `groupPolicy`, `dmPolicy`, `allowFrom`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- opdracht: `commands.native`, `commands.useAccessGroups` (globaal), `configWrites`, `slashCommand.ephemeral`
- Gateway: `proxy`
- antwoord/geschiedenis: `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- levering: `textChunkLimit` (standaard `2000`), `maxLinesPerMessage` (standaard `17`)
- streaming: `streaming.mode`, `streaming.chunkMode`, `streaming.preview.*`, `streaming.progress.*`, `streaming.block.*` (verouderde platte sleutels `streamMode`, `draftChunk`, `blockStreaming`, `blockStreamingCoalesce`, `chunkMode` worden door `openclaw doctor --fix` naar `streaming.*` gemigreerd)
- media: `mediaMaxMb` (beperkt uitgaande Discord-uploads, standaard `100`)
- acties: `actions.*`
- aanwezigheid: `activity`, `status`, `activityType`, `activityUrl`, `autoPresence.*`
- UI: `ui.components.accentColor`
- functies: `threadBindings`, `bindings[]` op het hoogste niveau (`type: "acp"`), `pluralkit`, `execApprovals`, `intents`, `agentComponents.enabled`, `agentComponents.ttlMs`, `activities`, `heartbeat`, `responsePrefix`

</Accordion>

### Discord Activities

Stel `channels.discord.activities` in zodat agents zelfstandige HTML-widgets kunnen plaatsen die in Discord worden geopend. Het blok is optioneel; wanneer het ontbreekt, registreert OpenClaw geen Activity-routes, tool of interactiehandler. Zie [Discord Activities](/channels/discord-activities) voor de configuratie van het Developer Portal, de tunnel, beveiliging en probleemoplossing.

- `activities.clientSecret`: OAuth2-clientgeheim voor de Discord-applicatie; valt terug op `DISCORD_CLIENT_SECRET`
- `activities.applicationId`: optionele Activity-applicatie-ID; standaard de botapplicatie-ID die bij het opstarten van de Gateway wordt verkregen

## Veiligheid en beheer

- Behandel bottokens als geheimen (`DISCORD_BOT_TOKEN` heeft de voorkeur in bewaakte omgevingen).
- Verleen Discord-machtigingen volgens het principe van minimale rechten.
- Als de implementatie of status van opdrachten verouderd is, start je de Gateway opnieuw en controleer je dit nogmaals met `openclaw channels status --probe`.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Discord Activities" icon="window" href="/channels/discord-activities">
    Start interactieve HTML-widgets in Discord.
  </Card>
  <Card title="Koppeling" icon="link" href="/nl/channels/pairing">
    Koppel een Discord-gebruiker aan de Gateway.
  </Card>
  <Card title="Groepen" icon="users" href="/nl/channels/groups">
    Gedrag van groepschats en toestaanlijsten.
  </Card>
  <Card title="Kanaalroutering" icon="route" href="/nl/channels/channel-routing">
    Routeer inkomende berichten naar agents.
  </Card>
  <Card title="Beveiliging" icon="shield" href="/nl/gateway/security">
    Dreigingsmodel en beveiligingsversterking.
  </Card>
  <Card title="Routering met meerdere agents" icon="sitemap" href="/nl/concepts/multi-agent">
    Wijs servers en kanalen toe aan agents.
  </Card>
  <Card title="Slash-opdrachten" icon="terminal" href="/nl/tools/slash-commands">
    Gedrag van systeemeigen opdrachten.
  </Card>
</CardGroup>
