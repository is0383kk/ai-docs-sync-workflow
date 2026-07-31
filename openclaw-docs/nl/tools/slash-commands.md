---
read_when:
    - Chatopdrachten gebruiken of configureren
    - Foutopsporing voor opdrachtroutering of machtigingen
    - Begrijpen hoe skill-opdrachten worden geregistreerd
sidebarTitle: Slash commands
summary: Alle beschikbare slashcommando's, richtlijnen en inline-sneltoetsen — configuratie, routering en gedrag per interface.
title: Slashcommando's
x-i18n:
    generated_at: "2026-07-27T05:54:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ee5ee5e46d632a54ea92dea7ca61046288bf1998d05b08396107bec90e646fff
    source_path: tools/slash-commands.md
    workflow: 16
---

De Gateway verwerkt opdrachten die als zelfstandige berichten worden verzonden en beginnen met `/`.
Bash-opdrachten die alleen op de host worden uitgevoerd, gebruiken `! <cmd>` (met `/bash <cmd>` als alias).

Wanneer een gesprek aan een ACP-sessie is gekoppeld, wordt normale tekst naar de ACP-
harness gerouteerd. Beheeropdrachten voor de Gateway blijven lokaal: `/acp ...` bereikt altijd
de OpenClaw-opdrachthandler, en `/status` plus `/unfocus` blijven lokaal wanneer
opdrachtafhandeling voor het oppervlak is ingeschakeld.

## Drie typen opdrachten

<CardGroup cols={3}>
  <Card title="Opdrachten" icon="terminal">
    Zelfstandige `/...`-berichten die door de Gateway worden verwerkt. Moeten als
    enige inhoud van het bericht worden verzonden.
  </Card>
  <Card title="Directieven" icon="sliders">
    `/think`, `/fast`, `/verbose`, `/trace`, `/reasoning`, `/elevated`,
    `/exec`, `/model`, `/queue` — worden uit het bericht verwijderd voordat het model
    het ziet. Slaan sessie-instellingen blijvend op wanneer ze afzonderlijk worden verzonden; fungeren als inline-aanwijzingen
    wanneer ze samen met andere tekst worden verzonden.
  </Card>
  <Card title="Inline-snelkoppelingen" icon="bolt">
    `/help`, `/commands`, `/status`, `/whoami` — worden onmiddellijk uitgevoerd en
    verwijderd voordat het model de resterende tekst ziet. Alleen geautoriseerde afzenders.
  </Card>
</CardGroup>

<AccordionGroup>
  <Accordion title="Details over het gedrag van directieven">
    - Directieven worden uit het bericht verwijderd voordat het model het ziet.
    - In berichten met **alleen directieven** (het bericht bevat uitsluitend directieven) worden ze
      blijvend in de sessie opgeslagen en wordt er met een bevestiging geantwoord.
    - In **normale chatberichten** met andere tekst fungeren ze als inline-aanwijzingen en
      worden sessie-instellingen **niet** blijvend opgeslagen.
    - Directieven zijn alleen van toepassing op **geautoriseerde afzenders**. Als `commands.allowFrom`
      is ingesteld, wordt uitsluitend die toelatingslijst gebruikt; anders is autorisatie afkomstig van
      kanaaltoelatingslijsten, koppeling en permanent afgedwongen toegangsgroepen. Bij niet-geautoriseerde
      afzenders worden directieven als platte tekst behandeld.
  </Accordion>
</AccordionGroup>

## Configuratie

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: true,
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw",
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<ParamField path="commands.text" type="boolean" default="true">
  Schakelt het parseren van `/...` in chatberichten in. Op oppervlakken zonder systeemeigen opdrachten
  (WhatsApp, WebChat, Signal, iMessage, Google Chat, Microsoft Teams) werken tekst-
  opdrachten zelfs wanneer dit is ingesteld op `false`.
</ParamField>

<ParamField path="commands.native" type='boolean | "auto"' default='"auto"'>
  Registreert systeemeigen opdrachten. Automatisch: ingeschakeld voor Discord/Telegram; uitgeschakeld voor Slack;
  genegeerd voor providers zonder systeemeigen ondersteuning. Overschrijf dit per kanaal met
  `channels.<provider>.commands.native`. Op Discord slaat `false` de registratie van slash-opdrachten
  over; eerder geregistreerde opdrachten kunnen zichtbaar blijven totdat ze worden verwijderd.
</ParamField>

<ParamField path="commands.nativeSkills" type='boolean | "auto"' default='"auto"'>
  Registreert Skills-opdrachten waar ondersteund als systeemeigen opdrachten. Automatisch: ingeschakeld voor
  Discord/Telegram; uitgeschakeld voor Slack. Overschrijf dit met
  `channels.<provider>.commands.nativeSkills`.
</ParamField>

<ParamField path="commands.bash" type="boolean" default="false">
  Schakelt `! <cmd>` in om shellopdrachten op de host uit te voeren (`/bash <cmd>`-alias). Vereist
  `tools.elevated`-toelatingslijsten.
</ParamField>

<ParamField path="commands.bashForegroundMs" type="number" default="2000">
  Hoe lang bash wacht voordat naar de achtergrondmodus wordt overgeschakeld (`0` schakelt
  onmiddellijk over naar de achtergrond).
</ParamField>

<ParamField path="commands.config" type="boolean" default="false">
  Schakelt `/config` in (leest/schrijft `openclaw.json`). Alleen voor de eigenaar.
</ParamField>

<ParamField path="commands.mcp" type="boolean" default="false">
  Schakelt `/mcp` in (leest/schrijft door OpenClaw beheerde MCP-configuratie onder `mcp.servers`). Alleen voor de eigenaar.
</ParamField>

<ParamField path="commands.plugins" type="boolean" default="false">
  Schakelt `/plugins` in (detectie/status van plugins plus installeren en in-/uitschakelen). Schrijfbewerkingen zijn alleen voor de eigenaar.
</ParamField>

<ParamField path="commands.debug" type="boolean" default="false">
  Schakelt `/debug` in (configuratie-overschrijvingen alleen tijdens runtime). Alleen voor de eigenaar.
</ParamField>

<ParamField path="commands.restart" type="boolean" default="true">
  Schakelt `/restart` en externe `SIGUSR1`-herstartverzoeken in.
</ParamField>

<ParamField path="commands.ownerAllowFrom" type="string[]">
  Expliciete toelatingslijst voor de eigenaar voor opdrachtoppervlakken die alleen voor de eigenaar zijn. Staat los van
  `commands.allowFrom` en toegang via DM-koppeling.
</ParamField>

<ParamField path="channels.<channel>.commands.enforceOwnerForCommands" type="boolean" default="false">
  Per kanaal: vereist de identiteit van de eigenaar voor opdrachten die alleen voor de eigenaar zijn. Wanneer `true`,
  moet de afzender overeenkomen met `commands.ownerAllowFrom` of het interne bereik `operator.admin`
  hebben. Een jokertekenvermelding `allowFrom` is **niet** voldoende.
</ParamField>

<ParamField path="commands.ownerDisplay" type='"raw" | "hash"'>
  Bepaalt hoe eigenaar-id's in de systeemprompt worden weergegeven.
</ParamField>

<ParamField path="commands.ownerDisplaySecret" type="string">
  HMAC-geheim dat wordt gebruikt wanneer `commands.ownerDisplay: "hash"`.
</ParamField>

<ParamField path="commands.allowFrom" type="object">
  Toelatingslijst per provider voor opdrachtautorisatie. Wanneer deze is geconfigureerd, is dit de
  **enige** autorisatiebron voor opdrachten en directieven. Gebruik `"*"` als
  algemene standaardwaarde; providerspecifieke sleutels overschrijven deze.
</ParamField>

## Opdrachtenlijst

Opdrachten zijn afkomstig uit drie bronnen:

- **Ingebouwde kernopdrachten:** `src/auto-reply/commands-registry.shared.ts`
- **Gegenereerde dockopdrachten:** `src/auto-reply/commands-registry.data.ts`
- **Pluginopdrachten:** aanroepen van plugin `registerCommand()`

De beschikbaarheid hangt af van configuratievlaggen, het kanaaloppervlak en geïnstalleerde/ingeschakelde
plugins.

### Kernopdrachten

<AccordionGroup>
  <Accordion title="Sessies en uitvoeringen">
    | Opdracht | Beschrijving |
    | --- | --- |
    | `/new [model]` | Archiveer de huidige sessie en start een nieuwe |
    | `/reset [soft [message]]` | Stel de huidige sessie ter plaatse opnieuw in. `soft` behoudt het transcript, verwijdert hergebruikte sessie-id's van de CLI-backend en voert het opstartproces opnieuw uit |
    | `/name <title>` | Geef de huidige sessie een naam of wijzig de naam. Laat de titel weg om de huidige naam en een suggestie te zien |
    | `/compact [instructions]` | Compacteer de sessiecontext. Zie [Compaction](/nl/concepts/compaction) |
    | `/stop` | Breek de huidige uitvoering af |
    | `/session idle <duration\|off>` | Beheer de vervaldatum wegens inactiviteit van threadkoppelingen |
    | `/session max-age <duration\|off>` | Beheer de vervaldatum wegens maximale leeftijd van threadkoppelingen |
    | `/export-session [path]` | Alleen voor de eigenaar. Exporteer de huidige sessie naar HTML binnen de werkruimte. Alias: `/export` |
    | `/export-trajectory [path]` | Exporteer een JSONL-trajectbundel voor de huidige sessie. Alias: `/trajectory` |

    Expliciete `/export-session`-paden vervangen bestaande bestanden binnen de
    werkruimte. Laat het pad weg om een bestandsnaam te genereren die botsingen voorkomt.

    <Note>
      De Control UI onderschept getypte `/new` om een nieuwe
      dashboardsessie te maken en ernaartoe over te schakelen, behalve wanneer `session.dmScope: "main"` is geconfigureerd
      en de huidige bovenliggende sessie de hoofdsessie van de agent is — in dat geval stelt `/new`
      de hoofdsessie ter plaatse opnieuw in. Getypte `/reset` voert nog steeds de interne
      reset van de Gateway uit. Gebruik `/model default` wanneer je een vastgezette
      modelselectie voor de sessie wilt wissen.
    </Note>

  </Accordion>

  <Accordion title="Model- en uitvoeringsbesturing">
    | Opdracht | Beschrijving |
    | --- | --- |
    | `/think <level\|default>` | Stel het denkniveau in of wis de sessie-overschrijving. Aliassen: `/thinking`, `/t` |
    | `/verbose on\|off\|full` | Schakel uitgebreide uitvoer in of uit. Alias: `/v` |
    | `/trace on\|off` | Schakel trace-uitvoer van plugins voor de huidige sessie in of uit |
    | `/fast [status\|auto\|on\|off\|default]` | Toon snelle modus, stel deze in of wis deze |
    | `/reasoning [on\|off\|stream]` | Schakel de zichtbaarheid van redeneringen in of uit. Alias: `/reason` |
    | `/elevated [on\|off\|ask\|full]` | Schakel de verhoogde modus in of uit. Alias: `/elev` |
    | `/exec host=<auto\|sandbox\|gateway\|node> security=<deny\|allowlist\|full> ask=<off\|on-miss\|always> node=<id>` | Toon uitvoeringsstandaarden of stel deze in |
    | `/login [codex\|openai\|openai-codex]` | Koppel de Codex/OpenAI-aanmelding vanuit een privéchat of Web UI-sessie. Alleen voor eigenaar/beheerder |
    | `/model [name\|#\|status]` | Toon het model of stel het in |
    | `/models [provider] [page] [limit=<n>\|all]` | Toon geconfigureerde providers of modellen waarvoor authenticatie beschikbaar is |
    | `/queue <mode>` | Beheer het wachtrijgedrag voor actieve uitvoeringen. Zie [Wachtrij](/nl/concepts/queue) en [Wachtrijsturing](/nl/concepts/queue-steering) |
    | `/steer <message>` | Voeg richtlijnen toe aan de actieve uitvoering. Alias: `/tell`. Zie [Sturen](/nl/tools/steer) |

    <AccordionGroup>
      <Accordion title="veiligheid van uitgebreid / trace / snel / redenering">
        - `/verbose` is bedoeld voor foutopsporing — houd dit bij normaal gebruik **uitgeschakeld**.
        - `/trace` toont alleen trace-/foutopsporingsregels die eigendom zijn van plugins; normale uitgebreide berichten blijven uitgeschakeld.
        - `/fast auto|on|off` slaat een sessie-overschrijving blijvend op; gebruik de optie `inherit` in de Sessions UI om deze te wissen.
        - `/fast` is providerspecifiek: OpenAI/Codex koppelen dit aan `service_tier=priority`; rechtstreekse Anthropic-verzoeken koppelen dit aan `service_tier=auto` of `standard_only`.
        - `/reasoning`, `/verbose` en `/trace` zijn riskant in groepsomgevingen — ze kunnen interne redeneringen of plugindiagnostiek onthullen. Houd ze uitgeschakeld in groepschats.

      </Accordion>
      <Accordion title="Details over het wisselen van model">
        - `/model` slaat het nieuwe model onmiddellijk blijvend op in de sessie.
        - Als de agent inactief is, gebruikt de volgende uitvoering het meteen.
        - Als er een uitvoering actief is, wordt de wisseling als in afwachting gemarkeerd en toegepast bij het volgende geldige nieuwe probeerpunt.

      </Accordion>
    </AccordionGroup>

  </Accordion>

  <Accordion title="Detectie en status">
    | Opdracht | Beschrijving |
    | --- | --- |
    | `/help` | Toon het korte helpoverzicht |
    | `/commands` | Toon de gegenereerde opdrachtencatalogus |
    | `/tools [compact\|verbose]` | Toon wat de huidige agent op dit moment kan gebruiken |
    | `/status` | Toon de uitvoerings-/runtimestatus, de actieve tijd van de Gateway en het systeem, de status van plugins en het providergebruik/-quotum |
    | `/status plugins` | Toon gedetailleerde pluginstatus: laadfouten, quarantaines, storingen van kanaalplugins, afhankelijkheidsproblemen en compatibiliteitsmeldingen. Vereist `commands.plugins: true` |
    | `/goal [status\|start\|edit\|pause\|resume\|complete\|block\|clear] ...` | Beheer het duurzame [doel](/nl/tools/goal) van de huidige sessie |
    | `/diagnostics [note]` | Ondersteuningsrapportage alleen voor de eigenaar. Vraagt elke keer om goedkeuring voor uitvoering |
    | `/openclaw <request>` | Voer de installatie- en reparatiehulp van OpenClaw uit vanuit een DM van de eigenaar |
    | `/tasks` | Toon actieve/recente achtergrondtaken voor de huidige sessie |
    | `/context [list\|detail\|map\|json]` | Leg uit hoe de context wordt samengesteld |
    | `/whoami` | Toon je afzender-id. Alias: `/id` |
    | `/usage off\|tokens\|full\|reset\|cost` | Beheer de gebruiksvoetnoot per antwoord (`reset`/`inherit`/`clear`/`default` wist de sessie-overschrijving zodat de geconfigureerde standaardwaarde opnieuw wordt overgenomen) of toon een lokaal kostenoverzicht |
  </Accordion>

  <Accordion title="Skills, toelatingslijsten, goedkeuringen">
    | Opdracht | Beschrijving |
    | --- | --- |
    | `/skill <name> [input]` | Voer een skill op naam uit |
    | `/learn [request]` | Stel één controleerbare skill op uit het huidige gesprek of benoemde bronnen via [Skill Workshop](/nl/tools/skill-workshop) |
    | `/allowlist [list\|add\|remove] ...` | Beheer vermeldingen in de toelatingslijst. Alleen tekst |
    | `/approve <id> <decision>` | Handel goedkeuringsverzoeken voor exec of plugins af |
    | `/btw <question>` | Stel een tussenvraag zonder de sessiecontext te wijzigen. Alias: `/side`. Zie [BTW](/nl/tools/btw) |
  </Accordion>

  <Accordion title="Subagents en ACP">
    | Opdracht | Beschrijving |
    | --- | --- |
    | `/subagents list\|log\|info` | Inspecteer uitvoeringen van subagents voor de huidige sessie |
    | `/acp spawn\|cancel\|steer\|close\|sessions\|status\|set-mode\|set\|cwd\|permissions\|timeout\|model\|reset-options\|doctor\|install\|help` | Beheer ACP-sessies en runtimeopties. Voor runtimebesturing is een externe eigenaar of interne Gateway-beheerdersidentiteit vereist |
    | `/focus <target>` | Koppel de huidige Discord-thread of het huidige Telegram-onderwerp aan een sessiedoel |
    | `/unfocus` | Verwijder de huidige threadkoppeling |
    | `/agents` | Geef aan threads gekoppelde agents voor de huidige sessie weer |
  </Accordion>

  <Accordion title="Schrijfbewerkingen en beheer alleen voor eigenaren">
    | Opdracht | Vereist | Beschrijving |
    | --- | --- | --- |
    | `/config show\|get\|set\|unset` | `commands.config: true` | Lees of schrijf `openclaw.json`. Alleen voor eigenaren |
    | `/mcp show\|get\|set\|unset` | `commands.mcp: true` | Lees of schrijf de door OpenClaw beheerde MCP-serverconfiguratie. Alleen voor eigenaren |
    | `/plugins list\|inspect\|show\|get\|install\|enable\|disable` | `commands.plugins: true` | Inspecteer of wijzig de pluginstatus. Schrijfbewerkingen alleen voor eigenaren. Alias: `/plugin` |
    | `/debug show\|set\|unset\|reset` | `commands.debug: true` | Configuratieoverschrijvingen die alleen tijdens runtime gelden. Alleen voor eigenaren |
    | `/restart` | `commands.restart: true` (standaard) | Start OpenClaw opnieuw |
    | `/send on\|off\|inherit` | eigenaar | Stel het verzendbeleid in |
  </Accordion>

  <Accordion title="Spraak, TTS, kanaalbesturing">
    | Opdracht | Beschrijving |
    | --- | --- |
    | `/tts on\|off\|status\|chat\|latest\|provider\|limit\|summary\|audio\|help` | Bestuur TTS. Zie [TTS](/nl/tools/tts) |
    | `/activation mention\|always` | Stel de groepsactiveringsmodus in |
    | `/bash <command>` | Voer een shellopdracht op de host uit. Alias: `! <command>`. Vereist `commands.bash: true` |
    | `!poll [sessionId]` | Controleer een bash-taak op de achtergrond |
    | `!stop [sessionId]` | Stop een bash-taak op de achtergrond |
  </Accordion>
</AccordionGroup>

### Dockopdrachten

Dockopdrachten schakelen de antwoordroute van de actieve sessie over naar een ander gekoppeld kanaal.
Zie [Kanaaldocking](/nl/concepts/channel-docking) voor configuratie en probleemoplossing.

Gegenereerd vanuit kanaalplugins met ondersteuning voor native opdrachten:

- `/dock-discord` (alias: `/dock_discord`)
- `/dock-mattermost` (alias: `/dock_mattermost`)
- `/dock-slack` (alias: `/dock_slack`)
- `/dock-telegram` (alias: `/dock_telegram`)

Dockopdrachten vereisen `session.identityLinks`. De afzender van de bron en de doelpeer
moeten deel uitmaken van dezelfde identiteitsgroep.

### Meegeleverde pluginopdrachten

| Opdracht                                                 | Beschrijving                                                                                                                                                                                    |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/dreaming [on\|off\|status\|help]`                     | Schakel Dreaming voor het geheugen in of uit (eigenaar of Gateway-beheerder). Zie [Dreaming](/nl/concepts/dreaming)                                                                                                            |
| `/pair [qr\|status\|pending\|approve\|cleanup\|notify]` | Beheer apparaatkoppeling. Zie [Koppeling](/nl/channels/pairing)                                                                                                                                        |
| `/phone status\|arm ...\|disarm`                        | Sta tijdelijk Node-opdrachten met een hoog risico toe (camera/scherm/computer/schrijfbewerkingen). Zie [Computergebruik](/nl/nodes/computer-use)                                                                               |
| `/voice status\|list\|set <voiceId>`                    | Beheer de spraakconfiguratie van Talk. Native naam in Discord: `/talkvoice`                                                                                                                                    |
| `/card ...`                                             | Verzend voorinstellingen voor uitgebreide LINE-kaarten. Zie [LINE](/nl/channels/line)                                                                                                                                        |
| `/codex <action> ...`                                   | Koppel, stuur en inspecteer het Codex-appserverharnas (status, threads, hervatten, model, snel, machtigingen, compact, controle, mcp, skills en meer). Zie [Codex-harnas](/nl/plugins/codex-harness) |

Alleen voor QQBot: `/bot-ping`, `/bot-version`, `/bot-help`, `/bot-upgrade`, `/bot-logs`

### Skillopdrachten

Door gebruikers aanroepbare skills worden beschikbaar gesteld als slash-opdrachten:

- `/skill <name> [input]` werkt altijd als algemeen toegangspunt.
- Skills kunnen als rechtstreekse opdrachten worden geregistreerd (bijvoorbeeld `/prose` voor OpenProse).
- De registratie van native skillopdrachten wordt bestuurd door `commands.nativeSkills` en
  `channels.<provider>.commands.nativeSkills`.
- Namen worden opgeschoond tot `a-z0-9_` (maximaal 32 tekens); bij botsingen worden numerieke achtervoegsels toegevoegd.

<AccordionGroup>
  <Accordion title="Routering van skillopdrachten">
    Skillopdrachten worden standaard als een normaal verzoek naar het model gerouteerd.

    Skills kunnen `command-dispatch: tool` declareren om rechtstreeks naar een tool te routeren
    (deterministisch, zonder betrokkenheid van het model). Voorbeeld: `/prose` (OpenProse-plugin)
    — zie [OpenProse](/nl/prose).

  </Accordion>
  <Accordion title="Argumenten van native opdrachten">
    Discord gebruikt automatisch aanvullen voor dynamische opties en knopmenu's wanneer vereiste
    argumenten zijn weggelaten. Telegram en Slack tonen een knopmenu voor opdrachten met
    keuzemogelijkheden. Dynamische keuzemogelijkheden worden bepaald aan de hand van het model van de doelsessie, zodat model-
    specifieke opties zoals `/think`-niveaus de `/model`-overschrijving van de sessie volgen.
  </Accordion>
</AccordionGroup>

## `/tools`: wat de agent nu kan gebruiken

`/tools` beantwoordt een runtimevraag: **wat deze agent op dit moment in dit
gesprek kan gebruiken** — niet een statische configuratiecatalogus.

```text
/tools         # compacte weergave
/tools verbose # met korte beschrijvingen
```

Resultaten gelden per sessie. Als je de agent, het kanaal, de thread, de autorisatie
van de afzender of het model wijzigt, kan de uitvoer veranderen. Gebruik voor het bewerken van profielen en overschrijvingen
het paneel Tools in de Control UI of configuratieoppervlakken.

## `/model`: modelselectie

```text
/model             # modelkiezer tonen
/model list        # hetzelfde
/model 3           # op nummer uit de kiezer selecteren
/model openai/gpt-5.4
/model opus@anthropic:default
/model default     # modelselectie van de sessie wissen
/model status      # gedetailleerde weergave met eindpunt en API-modus
```

In Discord openen `/model` en `/models` een interactieve kiezer met vervolgkeuzelijsten voor providers en
modellen. De kiezer respecteert `agents.defaults.modelPolicy.allow`,
inclusief `provider/*`-vermeldingen. Zonder een expliciete toelatingslijst beperken modelvermeldingen en
aliassen de selectie niet.

## `/config`: configuratie naar schijf schrijven

<Note>
  Alleen voor eigenaren. Standaard uitgeschakeld — schakel dit in met `commands.config: true`.
</Note>

```text
/config show
/config show channels.whatsapp.responsePrefix
/config get channels.whatsapp.responsePrefix
/config set channels.whatsapp.responsePrefix="[openclaw]"
/config unset channels.whatsapp.responsePrefix
```

De configuratie wordt vóór het schrijven gevalideerd. Ongeldige wijzigingen worden geweigerd. `/config`-
updates blijven na opnieuw starten behouden.

## `/mcp`: MCP-serverconfiguratie

<Note>
  Alleen voor eigenaren. Standaard uitgeschakeld — schakel dit in met `commands.mcp: true`.
</Note>

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

`/mcp` slaat de configuratie op in de OpenClaw-configuratie, niet in de projectinstellingen van de ingebedde agent.
`/mcp show` maakt velden met aanmeldgegevens, waarden van herkende vlaggen voor aanmeldgegevens
en bekende argumenten die op geheimen lijken onleesbaar. Wanneer de opdracht vanuit een groep wordt uitgevoerd, wordt de
configuratie privé naar de eigenaar verzonden; als er geen privéroute naar de eigenaar
beschikbaar is, wordt de opdracht veilig afgebroken en wordt de eigenaar gevraagd het opnieuw te proberen vanuit een rechtstreeks
gesprek.

## `/debug`: overschrijvingen die alleen tijdens runtime gelden

<Note>
  Alleen voor eigenaren. Standaard uitgeschakeld — schakel dit in met `commands.debug: true`.
  Overschrijvingen worden onmiddellijk toegepast op nieuwe configuratielezingen, maar schrijven **niet** naar schijf.
</Note>

```text
/debug show
/debug set channels.whatsapp.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset channels.whatsapp.responsePrefix
/debug reset
```

## `/plugins`: pluginbeheer

<Note>
  Schrijfbewerkingen alleen voor eigenaren. Standaard uitgeschakeld — schakel dit in met `commands.plugins: true`.
</Note>

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
/plugins install clawhub:<package>
/plugins install npm:@openclaw/<official-package>
/plugins install npm:<package> --force
/plugins install git:<repository>@<ref> --force
```

`/plugins enable|disable` werkt de pluginconfiguratie bij en herlaadt de
pluginruntime van de Gateway tijdens bedrijf voor nieuwe agentbeurten. `/plugins install` start beheerde
Gateways automatisch opnieuw omdat de bronmodules van de plugin zijn gewijzigd. Voor installaties uit vertrouwde ClawHub-
bronnen en de officiële catalogus is geen extra bevestiging nodig. Willekeurige npm-,
git-, archief-, `npm-pack:`- en lokale padbronnen tonen een herkomstwaarschuwing en
vereisen een afsluitende `--force` nadat je de bron hebt gecontroleerd. Deze vlag bevestigt
de bron en staat vervanging van een bestaande installatie toe; de vlag omzeilt
`security.installPolicy` of de beveiligingscontroles van het installatieprogramma niet. Voor ClawHub-releases met
risicowaarschuwingen blijft de afzonderlijke, uitsluitend via de shell beschikbare
vlag `--acknowledge-clawhub-risk` vereist. Marketplace-, gekoppelde en vastgezette installaties
blijven eveneens uitsluitend via de shell beschikbaar.

## `/trace`: uitvoer van plugintracering

```text
/trace          # huidige traceringsstatus tonen
/trace on
/trace off
```

`/trace` toont sessiespecifieke tracerings-/debugregels van plugins zonder de volledig uitgebreide
modus. Dit vervangt `/debug` (runtimeoverschrijvingen) of `/verbose` (normale
tooluitvoer) niet.

## `/btw`: tussenvragen

`/btw` is een snelle tussenvraag over de huidige sessiecontext. Alias: `/side`.

```text
/btw wat zijn we nu aan het doen?
/side wat is er veranderd terwijl de hoofduitvoering doorging?
```

In tegenstelling tot een normaal bericht:

- Gebruikt de huidige sessie als achtergrondcontext.
- Wordt in Codex-harnassessies uitgevoerd als een tijdelijke Codex-zijthread.
- Wijzigt toekomstige sessiecontext **niet**.
- Wordt niet naar de transcriptgeschiedenis geschreven.

Zie [BTW-tussenvragen](/nl/tools/btw) voor het volledige gedrag.

## Opmerkingen per oppervlak

<AccordionGroup>
  <Accordion title="Sessiebereik per oppervlak">
    - **Tekstopdrachten:** worden uitgevoerd in de normale chatsessie (privéberichten delen `main`, groepen hebben hun eigen sessie).
    - **Native Discord-opdrachten:** `agent:<agentId>:discord:slash:<userId>`
    - **Native Slack-opdrachten:** `agent:<agentId>:slack:slash:<userId>` (voorvoegsel configureerbaar via `channels.slack.slashCommand.sessionPrefix`)
    - **Native Telegram-opdrachten:** `telegram:slash:<userId>` (richten zich via `CommandTargetSessionKey` op de chatsessie)
    - **`/login codex`** verzendt apparaatkoppelingscodes alleen via privégesprekken of antwoordpaden van de Web UI. Bij aanroepen vanuit Telegram-groepen/-onderwerpen wordt de eigenaar gevraagd de bot in een privébericht te benaderen.
    - **`/stop`** richt zich op de actieve chatsessie om de huidige uitvoering af te breken.

  </Accordion>
  <Accordion title="Slack-specifieke details">
    `channels.slack.slashCommand` ondersteunt één opdracht in `/openclaw`-stijl.
    Maak met `commands.native: true` één Slack-slashopdracht per ingebouwde
    opdracht. Registreer `/agentstatus` (niet `/status`), omdat Slack
    `/status` reserveert. Tekst `/status` werkt nog steeds in Slack-berichten.
  </Accordion>
  <Accordion title="Snel pad en inline-snelkoppelingen">
    - Berichten die alleen een opdracht bevatten van afzenders op de toelatingslijst, worden onmiddellijk verwerkt (omzeilen wachtrij + model).
    - Inline-snelkoppelingen (`/help`, `/commands`, `/status`, `/whoami`) werken ook wanneer ze in normale berichten zijn opgenomen en worden verwijderd voordat het model de resterende tekst ziet.
    - Ongeautoriseerde berichten die alleen een opdracht bevatten, worden stilzwijgend genegeerd; inline `/...`-tokens worden als platte tekst behandeld.

  </Accordion>
  <Accordion title="Opmerkingen over argumenten">
    - Opdrachten accepteren een optionele `:` tussen de opdracht en de argumenten (`/think: high`, `/send: on`).
    - `/new <model>` accepteert een modelalias, `provider/model` of een providernaam (globale overeenkomst); als er geen overeenkomst is, wordt de tekst als de berichttekst behandeld.
    - `/allowlist add|remove` vereist `commands.config: true` en respecteert `configWrites` van het kanaal.

  </Accordion>
</AccordionGroup>

## Providergebruik en -status

- **Providergebruik/-quotum** (bijv. "Claude 80% resterend") wordt in `/status` weergegeven voor de huidige modelprovider wanneer gebruiksregistratie is ingeschakeld.
- **Token-/cacheregels** in `/status` kunnen terugvallen op de nieuwste gebruiksvermelding in het transcript wanneer de momentopname van de live sessie weinig gegevens bevat.
- **Uitvoering versus runtime:** `/status` rapporteert `Execution` voor het effectieve sandboxpad en `Runtime` voor wie de sessie uitvoert: `OpenClaw Default`, `OpenAI Codex`, een CLI-backend of een ACP-backend.
- **Tokens/kosten per antwoord:** beheerd door `/usage off|tokens|full`.
- `/model status` gaat over modellen/authenticatie/eindpunten, niet over gebruik.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Skills" href="/nl/tools/skills" icon="puzzle-piece">
    Hoe slashopdrachten van Skills worden geregistreerd en afgeschermd.
  </Card>
  <Card title="Skills maken" href="/nl/tools/creating-skills" icon="hammer">
    Bouw een Skill die zijn eigen slashopdracht registreert.
  </Card>
  <Card title="BTW" href="/nl/tools/btw" icon="comments">
    Zijvragen zonder de sessiecontext te wijzigen.
  </Card>
  <Card title="Sturen" href="/nl/tools/steer" icon="compass">
    Stuur de agent tijdens de uitvoering met `/steer`.
  </Card>
</CardGroup>
