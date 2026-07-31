---
read_when:
    - Codingharnassen uitvoeren via ACP
    - Gespreksgebonden ACP-sessies instellen op berichtenkanalen
    - Een gesprek in een berichtenkanaal koppelen aan een permanente ACP-sessie
    - Problemen oplossen met de ACP-backend, Plugin-koppeling of levering van voltooiingen
    - /acp-opdrachten bedienen vanuit de chat
sidebarTitle: ACP agents
summary: Voer externe codeerharnassen (Claude Code, Cursor, Gemini CLI, expliciete Codex ACP, OpenClaw ACP, OpenCode) uit via de ACP-backend
title: ACP-agents
x-i18n:
    generated_at: "2026-07-27T05:17:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc7f32ff927c7e949be1595f6aa00ed034a51185c6a6b1e0df01a242954667d1
    source_path: tools/acp-agents.md
    workflow: 16
---

[Agent Client Protocol (ACP)](https://agentclientprotocol.com/)-sessies stellen
OpenClaw in staat externe coderingsomgevingen (Claude Code, Cursor, Copilot, Droid,
OpenClaw ACP, OpenCode, Gemini CLI en andere ondersteunde ACPX-omgevingen)
uit te voeren via een ACP-back-endplugin. Elke gestarte sessie wordt bijgehouden als een
[achtergrondtaak](/nl/automation/tasks).

<Note>
**ACP is het pad voor externe omgevingen, niet het standaardpad voor Codex.** De systeemeigen
Codex-appserverplugin beheert de bedieningselementen voor `/codex ...` en de standaard
ingebedde runtime voor `openai/gpt-*` voor agentbeurten; ACP beheert de bedieningselementen voor `/acp ...`
en `sessions_spawn({ runtime: "acp" })`-sessies.

Als je Codex of Claude Code als externe MCP-client rechtstreeks verbinding wilt laten maken met
bestaande OpenClaw-kanaalgesprekken, gebruik je
[`openclaw mcp serve`](/nl/cli/mcp) in plaats van ACP.
</Note>

## Welke pagina heb ik nodig?

| Je wilt...                                                                                      | Gebruik dit                           | Opmerkingen                                                                                                                                                                 |
| ----------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Codex in het huidige gesprek koppelen of bedienen                                               | `/codex bind`, `/codex threads`       | Systeemeigen Codex-appserverpad wanneer de Plugin `codex` is ingeschakeld: gekoppelde chatantwoorden, doorsturen van afbeeldingen, model/snelheid/machtigingen, stoppen en bijsturen. ACP is een expliciete terugvaloptie |
| Claude Code, Gemini CLI, expliciete Codex ACP of een andere externe omgeving _via_ OpenClaw uitvoeren | Deze pagina                      | Aan chats gekoppelde sessies, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, achtergrondtaken, runtimebediening                                                             |
| Een OpenClaw Gateway-sessie _als_ ACP-server beschikbaar stellen voor een editor of client      | [`openclaw acp`](/nl/cli/acp)            | Brugmodus: een IDE/client communiceert via stdio/WebSocket met ACP naar OpenClaw                                                                                            |
| Een lokale AI-CLI hergebruiken als terugvalmodel voor alleen tekst                             | [CLI-back-ends](/nl/gateway/cli-backends) | Geen ACP: geen OpenClaw-tools, geen ACP-bedieningselementen, geen omgevingsruntime                                                                                           |

## Werkt dit direct?

Ja, na installatie van de officiële ACP-runtimeplugin:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Broncodecheck-outs kunnen de lokale werkruimteplugin `extensions/acpx` gebruiken na
`pnpm install`. Voer `/acp doctor` uit voor een gereedheidscontrole.

OpenClaw leert agents alleen over het starten van ACP wanneer ACP **daadwerkelijk bruikbaar** is:
ACP moet zijn ingeschakeld, dispatch mag niet zijn uitgeschakeld, de huidige sessie mag
niet door de sandbox zijn geblokkeerd en er moet een gezonde runtimeback-end zijn geladen. Als
aan een voorwaarde niet wordt voldaan, blijven ACP-Skills en de ACP-richtlijnen voor `sessions_spawn`
verborgen, zodat de agent geen niet-beschikbare back-end voorstelt.

<AccordionGroup>
  <Accordion title="Aandachtspunten bij de eerste uitvoering">
    - Als `plugins.allow` is ingesteld, vormt dit een beperkende Plugin-inventaris en **moet** deze `acpx` bevatten, anders wordt de geïnstalleerde ACP-back-end opzettelijk geblokkeerd (`/acp doctor` meldt het ontbrekende item in de toelatingslijst).
    - De Codex ACP-adapter wordt geleverd met de Plugin `acpx` en wordt waar mogelijk lokaal gestart.
    - Codex ACP wordt uitgevoerd met een geïsoleerde `CODEX_HOME`. OpenClaw kopieert vertrouwde projectvertrouwensvermeldingen en veilige routeringsconfiguratie voor modellen/providers (`model`, `model_provider`, `model_reasoning_effort`, `sandbox_mode` en veilige velden van `model_providers.<name>`) uit de Codex-hostconfiguratie; authenticatie, meldingen en hooks blijven uitsluitend in de hostconfiguratie.
    - Andere adapters voor doelomgevingen kunnen bij het eerste gebruik op aanvraag worden opgehaald met `npx`.
    - Authenticatie bij de leverancier moet voor die omgeving al op de host bestaan.
    - Als de host geen npm- of netwerktoegang heeft, mislukt het ophalen van adapters bij de eerste uitvoering totdat caches vooraf zijn gevuld of de adapter op een andere manier is geïnstalleerd.

  </Accordion>
  <Accordion title="Runtimevereisten">
    ACP start een echt extern omgevingsproces. OpenClaw beheert routering,
    de status van achtergrondtaken, aflevering, koppelingen en beleid; de omgeving beheert
    de eigen provideraanmelding, modelcatalogus, het bestandssysteemgedrag en de systeemeigen tools.

    Controleer het volgende voordat je OpenClaw de schuld geeft:

    - `/acp doctor` meldt een ingeschakelde, gezonde back-end.
    - De doel-id is toegestaan door `acp.allowedAgents` wanneer die toelatingslijst is ingesteld.
    - De omgevingsopdracht kan op de Gateway-host worden gestart.
    - Providerauthenticatie is aanwezig voor die omgeving (`claude`, `codex`, `gemini`, `opencode`, `droid` enzovoort).
    - Het geselecteerde model bestaat voor die omgeving - model-id's zijn niet overdraagbaar tussen omgevingen.
    - De aangevraagde `cwd` bestaat en is toegankelijk; laat anders `cwd` weg en laat de back-end de standaardwaarde gebruiken.
    - De machtigingsmodus past bij het werk. Niet-interactieve sessies kunnen niet op systeemeigen machtigingsprompts klikken, dus coderingsuitvoeringen met veel schrijf- of uitvoeracties vereisen doorgaans een ACPX-machtigingsprofiel dat zonder interactie kan doorgaan.

  </Accordion>
</AccordionGroup>

OpenClaw-plugintools en ingebouwde OpenClaw-tools worden standaard **niet**
beschikbaar gesteld aan ACP-omgevingen. Schakel de expliciete MCP-bruggen in
[ACP-agents - configuratie](/nl/tools/acp-agents-setup) alleen in wanneer de omgeving
die tools rechtstreeks moet aanroepen.

## Ondersteunde omgevingsdoelen

Gebruik met de back-end `acpx` deze id's als doelen voor `/acp spawn <id>` of
`sessions_spawn({ runtime: "acp", agentId: "<id>" })`:

| Omgevings-id | Gebruikelijke back-end                          | Opmerkingen                                                                        |
| ------------ | ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `claude`     | Claude Code ACP-adapter                        | Vereist Claude Code-authenticatie op de host.                                      |
| `codex`      | Codex ACP-adapter                              | Alleen een expliciete ACP-terugvaloptie wanneer systeemeigen `/codex` niet beschikbaar is of ACP wordt aangevraagd. |
| `copilot`    | GitHub Copilot ACP-adapter                     | Vereist authenticatie voor de Copilot-CLI/runtime.                                 |
| `cursor`     | Cursor CLI ACP (`cursor-agent acp`)            | Overschrijf de acpx-opdracht als een lokale installatie een ander ACP-ingangspunt beschikbaar stelt. |
| `droid`      | Factory Droid CLI                              | Vereist Factory/Droid-authenticatie of `FACTORY_API_KEY` in de omgevingsomgeving.  |
| `fast-agent` | fast-agent-mcp ACP-adapter                     | Wordt op aanvraag opgehaald met `uvx`.                                |
| `gemini`     | Gemini CLI ACP-adapter                         | Vereist Gemini CLI-authenticatie of configuratie van een API-sleutel.              |
| `iflow`      | iFlow CLI                                      | De beschikbaarheid van de adapter en modelbediening hangen af van de geïnstalleerde CLI. |
| `kilocode`   | Kilo Code CLI                                  | De beschikbaarheid van de adapter en modelbediening hangen af van de geïnstalleerde CLI. |
| `kimi`       | Kimi/Moonshot CLI                              | Vereist Kimi/Moonshot-authenticatie op de host.                                    |
| `kiro`       | Kiro CLI                                       | De beschikbaarheid van de adapter en modelbediening hangen af van de geïnstalleerde CLI. |
| `mux`        | Mux CLI ACP-adapter                            | Wordt op aanvraag opgehaald met `npx`.                                |
| `opencode`   | OpenCode ACP-adapter                           | Vereist OpenCode CLI-/providerauthenticatie.                                       |
| `openclaw`   | OpenClaw Gateway-brug via `openclaw acp` | Hiermee kan een ACP-compatibele omgeving terugcommuniceren met een OpenClaw Gateway-sessie. |
| `qoder`      | Qoder CLI                                      | De beschikbaarheid van de adapter en modelbediening hangen af van de geïnstalleerde CLI. |
| `qwen`       | Qwen Code / Qwen CLI                           | Vereist Qwen-compatibele authenticatie op de host.                                 |
| `trae`       | Trae CLI ACP-adapter                           | De beschikbaarheid van de adapter en modelbediening hangen af van de geïnstalleerde CLI. |

`pi` (pi-acp) is ook geregistreerd in de acpx-back-end, maar is niet in
dezelfde betekenis een coderingsomgeving als de andere hierboven.

Aangepaste acpx-agentaliases kunnen in acpx zelf worden geconfigureerd, maar het OpenClaw-
beleid controleert vóór dispatch nog steeds `acp.allowedAgents` en eventuele
`agents.entries.*.runtime.acp.agent`-toewijzingen.

## Draaiboek voor operators

Snelle `/acp`-workflow vanuit de chat:

<Steps>
  <Step title="Starten">
    `/acp spawn claude --bind here`,
    `/acp spawn gemini --mode persistent --thread auto` of expliciet
    `/acp spawn codex --bind here`.
  </Step>
  <Step title="Werken">
    Ga verder in het gekoppelde gesprek of de gekoppelde thread (of richt je expliciet
    op de sessiesleutel).
  </Step>
  <Step title="Status controleren">
    `/acp status`
  </Step>
  <Step title="Afstemmen">
    `/acp model <provider/model>`, `/acp permissions <profile>`,
    `/acp timeout <seconds>`.
  </Step>
  <Step title="Bijsturen">
    Zonder de context te vervangen: `/acp steer tighten logging and continue`.
  </Step>
  <Step title="Stoppen">
    `/acp cancel` (huidige beurt) of `/acp close` (sessie + koppelingen).
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Details van de levenscyclus">
    - Spawnen maakt een ACP-runtimesessie of hervat deze, legt ACP-metadata vast in het OpenClaw-sessiearchief en kan een achtergrondtaak maken wanneer de run eigendom is van de bovenliggende taak.
    - ACP-sessies die eigendom zijn van de bovenliggende taak, worden als achtergrondwerk behandeld, zelfs wanneer de runtimesessie persistent is; voltooiing en levering tussen oppervlakken verlopen via de taakmelder van de bovenliggende taak, in plaats van als een normale, voor de gebruiker zichtbare chatsessie.
    - Taakonderhoud sluit beëindigde of verweesde eenmalige ACP-sessies die eigendom zijn van een bovenliggende taak. Persistente ACP-sessies blijven behouden zolang er een actieve gespreksbinding bestaat; verouderde persistente sessies zonder actieve binding worden gesloten, zodat ze niet stilzwijgend kunnen worden hervat nadat de bijbehorende taak is voltooid of de taakregistratie ervan is verdwenen.
    - Gebonden vervolgberichten gaan rechtstreeks naar de ACP-sessie totdat de binding wordt gesloten, de focus verliest, wordt gereset of verloopt.
    - Gateway-opdrachten blijven lokaal. `/acp ...`, `/status` en `/unfocus` worden nooit als normale prompttekst naar een gebonden ACP-harnas verzonden.
    - `cancel` breekt de actieve beurt af wanneer de backend annulering ondersteunt; de binding of sessiemetadata wordt niet verwijderd.
    - `close` beëindigt de ACP-sessie vanuit het perspectief van OpenClaw en verwijdert de binding. Een harnas kan zijn eigen bovenliggende geschiedenis blijven bewaren als het hervatten ondersteunt.
    - De acpx-Plugin ruimt na `close` de processtructuren van wrappers en adapters op die eigendom zijn van OpenClaw, en ruimt tijdens het opstarten van de Gateway verouderde, verweesde ACPX-processen op die eigendom zijn van OpenClaw.
    - Inactieve runtimeworkers komen na de ingebouwde inactiviteitsperiode in aanmerking voor opruiming; opgeslagen sessiemetadata blijft beschikbaar voor `/acp sessions`.

  </Accordion>
  <Accordion title="Routeringsregels voor native Codex">
    Triggers in natuurlijke taal die naar de **native Codex-Plugin** moeten routeren
    wanneer deze is ingeschakeld:

    - "Bind dit Discord-kanaal aan Codex."
    - "Koppel deze chat aan Codex-thread `<id>`."
    - "Toon Codex-threads en bind vervolgens deze."

    Native Codex-gespreksbinding is het standaardpad voor chatbesturing.
    Dynamische OpenClaw-tools worden nog steeds via OpenClaw uitgevoerd, terwijl native Codex-
    tools zoals shell/apply-patch binnen Codex worden uitgevoerd. Voor native Codex-
    toolgebeurtenissen injecteert OpenClaw per beurt een native hookrelay, zodat Plugin-hooks
    `before_tool_call` kunnen blokkeren, `after_tool_call` kunnen observeren en Codex-
    gebeurtenissen van `PermissionRequest` via OpenClaw-goedkeuringen kunnen routeren. Codex-hooks voor `Stop`
    worden doorgestuurd naar OpenClaw `before_agent_finalize`, waar Plugins
    nog één modeldoorgang kunnen aanvragen voordat Codex het antwoord voltooit. De relay blijft
    bewust conservatief: deze wijzigt geen argumenten van native Codex-tools
    en herschrijft geen Codex-threadregistraties. Gebruik expliciete ACP alleen wanneer je het
    ACP-runtime-/sessiemodel wilt. De ondersteuningsgrens voor ingebedde Codex is
    gedocumenteerd in het
    [ondersteuningscontract voor Codex-harnas v1](/nl/plugins/codex-harness-runtime#v1-support-contract).

  </Accordion>
  <Accordion title="Overzicht voor model-, provider- en runtimeselectie">
    - verouderde Codex-modelreferenties - verouderde modelroute voor Codex OAuth/abonnement, hersteld door doctor.
    - `openai/*` - ingebedde native Codex-app-serverruntime voor OpenAI-agentbeurten.
    - `/codex ...` - native Codex-gespreksbesturing.
    - `/acp ...` of `runtime: "acp"` - expliciete ACP-/acpx-besturing.

  </Accordion>
  <Accordion title="Triggers in natuurlijke taal voor ACP-routering">
    Triggers die naar de ACP-runtime moeten routeren:

    - "Voer dit uit als een eenmalige Claude Code ACP-sessie en vat het resultaat samen."
    - "Gebruik Gemini CLI voor deze taak in een thread en houd vervolgberichten daarna in diezelfde thread."
    - "Voer Codex via ACP uit in een achtergrondthread."

    OpenClaw kiest `runtime: "acp"`, bepaalt het harnas `agentId`, bindt waar ondersteund
    aan het huidige gesprek of de huidige thread en routeert vervolgberichten
    naar die sessie totdat deze wordt gesloten of verloopt. Codex volgt dit pad alleen wanneer
    ACP/acpx expliciet is opgegeven of de native Codex-Plugin niet beschikbaar is voor de
    gevraagde bewerking.

    Voor `sessions_spawn` wordt `runtime: "acp"` alleen aangeboden wanneer ACP is
    ingeschakeld, de aanvrager niet in een sandbox wordt uitgevoerd en een ACP-runtimebackend is
    geladen. `acp.dispatch.enabled=false` pauzeert automatische ACP-threadverzending,
    maar verbergt of blokkeert expliciete aanroepen van `sessions_spawn({ runtime: "acp" })`
    niet. Het is gericht op ACP-harnas-id's zoals `codex`, `claude`, `droid`,
    `gemini` of `opencode`. Geef geen normale agent-id uit de OpenClaw-configuratie
    van `agents_list` door, tenzij die vermelding expliciet is geconfigureerd met
    `agents.entries.*.runtime.type="acp"`; gebruik anders de standaardruntime voor sub-agents.
    Wanneer een OpenClaw-agent is geconfigureerd met
    `runtime.type="acp"`, gebruikt OpenClaw `runtime.acp.agent` als de onderliggende
    harnas-id.

  </Accordion>
</AccordionGroup>

## ACP versus sub-agents

Gebruik ACP wanneer je een externe harnasruntime wilt. Gebruik de **native Codex-
app-server** voor Codex-gespreksbinding en -besturing wanneer de Plugin `codex`
is ingeschakeld. Gebruik **sub-agents** wanneer je gedelegeerde runs wilt die native in OpenClaw zijn.

| Onderdeel     | ACP-sessie                            | Sub-agent-run                       |
| ------------- | ------------------------------------- | ----------------------------------- |
| Runtime       | ACP-backend-Plugin (bijvoorbeeld acpx) | Native OpenClaw-runtime voor sub-agents |
| Sessiesleutel | `agent:<agentId>:acp:<uuid>`                    | `agent:<agentId>:subagent:<uuid>`                  |
| Hoofdopdrachten | `/acp ...`                  | `/subagents ...`                  |
| Spawntool     | `sessions_spawn` met `runtime:"acp"` | `sessions_spawn` (standaardruntime) |

Zie ook [Sub-agents](/nl/tools/subagents).

## Hoe ACP Claude Code uitvoert

Voor Claude Code via ACP bestaat de stack uit:

1. Besturingslaag voor OpenClaw ACP-sessies.
2. Officiële runtime-Plugin `@openclaw/acpx`.
3. Claude ACP-adapter.
4. Runtime-/sessiemechanismen aan de Claude-zijde.

ACP Claude is een **harnassessie** met ACP-besturing, sessiehervatting,
registratie van achtergrondtaken en optionele gespreks-/threadbinding.

CLI-backends zijn afzonderlijke, uitsluitend tekstuele lokale fallbackruntimes - zie
[CLI-backends](/nl/gateway/cli-backends).

Voor operators geldt in de praktijk:

- **Wil je `/acp spawn`, bindbare sessies, runtimebesturing of persistent harnaswerk?** Gebruik ACP.
- **Wil je een eenvoudige lokale tekstfallback via de onbewerkte CLI?** Gebruik CLI-backends.

## Gebonden sessies

### Mentaal model

- **Chatoppervlak** - waar mensen blijven praten (Discord-kanaal, Telegram-onderwerp, iMessage-chat).
- **ACP-sessie** - de duurzame Codex-/Claude-/Gemini-runtimestatus waarnaar OpenClaw routeert.
- **Onderliggende thread/onderwerp** - een optioneel extra berichtenoppervlak dat alleen door `--thread ...` wordt gemaakt.
- **Runtimewerkruimte** - de bestandssysteemlocatie (`cwd`, repo-check-out, backendwerkruimte) waar het harnas wordt uitgevoerd. Onafhankelijk van het chatoppervlak.

### Bindingen aan het huidige gesprek

`/acp spawn <harness> --bind here` koppelt het huidige gesprek vast aan de
gespawnde ACP-sessie - geen onderliggende thread, hetzelfde chatoppervlak. OpenClaw blijft
verantwoordelijk voor transport, authenticatie, veiligheid en levering. Vervolgberichten in dat
gesprek worden naar dezelfde sessie gerouteerd; `/new` en `/reset` resetten de sessie
ter plaatse; `/acp close` verwijdert de binding.

Voorbeelden:

```text
/codex bind                                              # native Codex-binding, routeer toekomstige berichten hierheen
/codex model gpt-5.4                                     # stel de gebonden native Codex-thread af
/codex stop                                              # bestuur de actieve native Codex-beurt
/acp spawn codex --bind here                             # expliciete ACP-fallback voor Codex
/acp spawn codex --thread auto                           # kan een onderliggende thread/onderwerp maken en daaraan binden
/acp spawn codex --bind here --cwd /workspace/repo       # dezelfde chatbinding, Codex wordt uitgevoerd in /workspace/repo
```

<AccordionGroup>
  <Accordion title="Bindingsregels en exclusiviteit">
    - `--bind here` en `--thread ...` sluiten elkaar uit.
    - `--bind here` werkt alleen op kanalen die binding aan het huidige gesprek aanbieden; anders retourneert OpenClaw een duidelijk bericht dat dit niet wordt ondersteund. Bindingen blijven behouden na herstarts van de Gateway.
    - Op Discord bepaalt `spawnSessions` of onderliggende threads voor `--thread auto|here` kunnen worden gemaakt - niet voor `--bind here`.
    - Als je zonder `--cwd` naar een andere ACP-agent spawnt, neemt OpenClaw standaard de werkruimte van de **doelagent** over. Ontbrekende overgenomen paden (`ENOENT`/`ENOTDIR`) vallen terug op de backendstandaard; andere toegangsfouten (bijvoorbeeld `EACCES`) worden als spawnfouten weergegeven.
    - Gateway-beheeropdrachten blijven lokaal in gebonden gesprekken - opdrachten van `/acp ...` worden door OpenClaw verwerkt, zelfs wanneer normale vervolgtekst naar de gebonden ACP-sessie wordt gerouteerd; `/status` en `/unfocus` blijven ook lokaal wanneer opdrachtverwerking voor dat oppervlak is ingeschakeld.

  </Accordion>
  <Accordion title="Aan threads gebonden sessies">
    Wanneer threadbindingen zijn ingeschakeld voor een kanaaladapter:

    - OpenClaw bindt een thread aan een ACP-doelsessie.
    - Vervolgberichten in die thread worden naar de gebonden ACP-sessie gerouteerd.
    - ACP-uitvoer wordt teruggeleverd aan dezelfde thread.
    - Focusverlies/sluiten/archiveren/inactiviteitstime-out of het verstrijken van de maximale leeftijd verwijdert de binding.
    - `/acp close`, `/acp cancel`, `/acp status`, `/status` en `/unfocus` zijn Gateway-opdrachten, geen prompts voor het ACP-harnas.

    Vereiste functievlaggen voor threadgebonden ACP:

    - `acp.enabled=true`
    - `acp.dispatch.enabled` is standaard ingeschakeld (stel `false` in om automatische ACP-threadverzending te pauzeren; expliciete aanroepen van `sessions_spawn({ runtime: "acp" })` blijven werken).
    - Het spawnen van threadsessies door kanaaladapters is ingeschakeld (standaard: `true`):
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`

    Ondersteuning voor threadbinding is adapterspecifiek. Als de actieve kanaaladapter
    geen threadbindingen ondersteunt, retourneert OpenClaw een duidelijk bericht
    dat dit niet wordt ondersteund of niet beschikbaar is.

  </Accordion>
  <Accordion title="Kanalen met threadondersteuning">
    - Elke kanaaladapter die mogelijkheden voor sessie-/threadbinding beschikbaar stelt.
    - Huidige ingebouwde ondersteuning: **Discord**-threads/-kanalen, **Telegram**-onderwerpen (forumonderwerpen in groepen/supergroepen en DM-onderwerpen).
    - Plugin-kanalen kunnen via dezelfde bindingsinterface ondersteuning toevoegen.

  </Accordion>
</AccordionGroup>

## Persistente kanaalbindingen

Configureer voor niet-tijdelijke workflows persistente ACP-bindingen in
vermeldingen van `bindings[]` op het hoogste niveau.

### Bindingsmodel

<ParamField path="bindings[].type" type='"acp"'>
  Markeert een persistente ACP-gespreksbinding.
</ParamField>
<ParamField path="bindings[].match" type="object">
  Identificeert het doelgesprek. Vormen per kanaal:

- **Discord-kanaal/-thread:** `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
- **Slack-kanaal/DM:** `match.channel="slack"` + `match.peer.id="<channelId|channel:<channelId>|#<channelId>|userId|user:<userId>|slack:<userId>|<@userId>>"`. Geef de voorkeur aan stabiele Slack-id's; kanaalkoppelingen komen ook overeen met antwoorden in de threads van dat kanaal.
- **Telegram-forumonderwerp:** `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
- **WhatsApp-DM/-groep:** `match.channel="whatsapp"` + `match.peer.id="<E.164|group JID>"`. Gebruik E.164-nummers zoals `+15555550123` voor rechtstreekse chats en WhatsApp-groeps-JID's zoals `120363424282127706@g.us` voor groepen.
- **iMessage-DM/-groep:** `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`. Geef de voorkeur aan `chat_id:*` voor stabiele groepskoppelingen.

</ParamField>
<ParamField path="bindings[].agentId" type="string">
  De id van de OpenClaw-agent die eigenaar is.
</ParamField>
<ParamField path="bindings[].acp.mode" type='"persistent" | "oneshot"'>
  Optionele ACP-overschrijving.
</ParamField>
<ParamField path="bindings[].acp.label" type="string">
  Optioneel label voor de operator.
</ParamField>
<ParamField path="bindings[].acp.cwd" type="string">
  Optionele werkmap van de runtime.
</ParamField>
<ParamField path="bindings[].acp.backend" type="string">
  Optionele backendoverschrijving.
</ParamField>

### Standaardwaarden voor de runtime per agent

Gebruik `agents.entries.*.runtime` om de ACP-standaardwaarden eenmaal per agent te definiëren:

- `agents.entries.*.runtime.type="acp"`
- `agents.entries.*.runtime.acp.agent` (harness-id, bijvoorbeeld `codex` of `claude`)
- `agents.entries.*.runtime.acp.backend`
- `agents.entries.*.runtime.acp.mode`
- `agents.entries.*.runtime.acp.cwd`

**Voorrangsvolgorde van overschrijvingen voor aan ACP gekoppelde sessies:**

1. `bindings[].acp.*`
2. `agents.entries.*.runtime.acp.*`
3. Algemene ACP-standaardwaarden (bijvoorbeeld `acp.backend`)

### Voorbeeld

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
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
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
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

### Gedrag

- OpenClaw zorgt ervoor dat de geconfigureerde ACP-sessie bestaat nadat de kanaalspecifieke toelating is voltooid en voordat de sessie wordt gebruikt.
- Berichten in dat kanaal, onderwerp of die chat worden naar de geconfigureerde ACP-sessie gerouteerd.
- Geconfigureerde ACP-koppelingen beheren hun sessieroute. De fan-out van kanaaluitzendingen vervangt voor een overeenkomende koppeling niet de geconfigureerde ACP-sessie.
- In gekoppelde gesprekken stellen `/new` en `/reset` dezelfde ACP-sessiesleutel ter plaatse opnieuw in.
- Tijdelijke runtimekoppelingen (bijvoorbeeld gemaakt door flows voor threadfocus) blijven van toepassing waar ze aanwezig zijn.
- Bij ACP-starts tussen agents zonder expliciete `cwd` neemt OpenClaw de werkruimte van de doelagent over uit de agentconfiguratie.
- Ontbrekende overgenomen werkruimtepaden vallen terug op de standaard-cwd van de backend; toegangsfouten voor bestaande paden worden als startfouten weergegeven.

## ACP-sessies starten

Er zijn twee manieren om een ACP-sessie te starten:

<Tabs>
  <Tab title="Vanuit sessions_spawn">
    Gebruik `runtime: "acp"` om een ACP-sessie vanuit een agentbeurt of
    toolaanroep te starten.

    ```json
    {
      "task": "Open de repository en vat de mislukte tests samen",
      "runtime": "acp",
      "agentId": "codex",
      "thread": true,
      "mode": "session"
    }
    ```

    <Note>
    `runtime` is standaard `subagent`, dus stel `runtime: "acp"` expliciet in voor
    ACP-sessies. Als `agentId` wordt weggelaten, gebruikt OpenClaw `acp.defaultAgent`
    wanneer dit is geconfigureerd. `mode: "session"` vereist `thread: true` om een
    permanent gekoppeld gesprek te behouden.
    </Note>

  </Tab>
  <Tab title="Vanuit de opdracht /acp">
    Gebruik `/acp spawn` voor expliciete bediening door de operator vanuit de chat.

    ```text
    /acp spawn codex --mode persistent --thread auto
    /acp spawn codex --mode oneshot --thread off
    /acp spawn codex --bind here
    /acp spawn codex --thread here
    ```

    Belangrijkste vlaggen:

    - `--mode persistent|oneshot`
    - `--bind here|off`
    - `--thread auto|here|off`
    - `--cwd <absolute-path>`
    - `--label <name>`

    Zie [Slash-opdrachten](/nl/tools/slash-commands).

  </Tab>
</Tabs>

### Parameters van `sessions_spawn`

<ParamField path="task" type="string" required>
  Initiële prompt die naar de ACP-sessie wordt verzonden.
</ParamField>
<ParamField path="runtime" type='"acp"' required>
  Moet voor ACP-sessies `"acp"` zijn.
</ParamField>
<ParamField path="agentId" type="string">
  Id van de ACP-doelharness. Valt terug op `acp.defaultAgent` als die is ingesteld.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  Vraag waar ondersteund om de flow voor threadkoppeling.
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  `"run"` is eenmalig; `"session"` is permanent. Als `thread: true` is ingesteld en
  `mode` wordt weggelaten, kan OpenClaw afhankelijk van het
  runtimepad standaard permanent gedrag gebruiken. `mode: "session"` vereist `thread: true`.
</ParamField>
<ParamField path="cwd" type="string">
  Aangevraagde werkmap van de runtime (gevalideerd door het backend-/runtimebeleid).
  Als deze wordt weggelaten, neemt een ACP-start de werkruimte van de doelagent over wanneer die is geconfigureerd;
  ontbrekende overgenomen paden vallen terug op de standaardwaarden van de backend, terwijl daadwerkelijke
  toegangsfouten worden geretourneerd.
</ParamField>
<ParamField path="label" type="string">
  Label voor de operator dat in sessie-/bannertekst wordt gebruikt.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  Hervat een bestaande ACP-sessie in plaats van een nieuwe te maken. De agent
  speelt zijn gespreksgeschiedenis opnieuw af via `session/load`. Vereist
  `runtime: "acp"`.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  `"parent"` streamt voortgangssamenvattingen van de eerste ACP-run terug naar de aanvragende
  sessie als systeemgebeurtenissen. OpenClaw slaat de volledige relaygeschiedenis op in de
  SQLite-status van de onderliggende agent en verwijdert deze samen met de onderliggende sessie. Voortgangsstreams
  naar de bovenliggende sessie tonen standaard commentaar van de assistent en ACP-statusvoortgang, tenzij
  `streaming.progress.commentary=false`. Discord gebruikt voor previews naar de bovenliggende sessie ook standaard
  de voortgangsmodus wanneer geen streammodus is geconfigureerd. Statusvoortgang
  respecteert nog steeds `acp.stream.tagVisibility`, zodat tags zoals `plan`
  verborgen blijven tenzij ze expliciet zijn ingeschakeld.
</ParamField>

ACP-runs met `sessions_spawn` gebruiken `agents.defaults.subagents.runTimeoutSeconds`
voor hun standaardlimiet voor onderliggende beurten. De tool accepteert geen timeoutoverschrijvingen
per aanroep (`runTimeoutSeconds`/`timeoutSeconds` worden afgewezen met een foutmelding
dat de standaardwaarde moet worden geconfigureerd).

<ParamField path="model" type="string">
  Expliciete modeloverschrijving voor de onderliggende ACP-sessie. Codex ACP-starts
  normaliseren OpenAI-verwijzingen zoals `openai/gpt-5.4` naar de Codex ACP-opstartconfiguratie
  vóór `session/new`; slashvormen zoals `openai/gpt-5.4/high` stellen ook
  de redeneerinspanning van Codex ACP in. Wanneer weggelaten, gebruikt `sessions_spawn({ runtime: "acp" })`
  bestaande standaardmodellen voor subagents (`agents.defaults.subagents.model` of
  `agents.entries.*.subagents.model`) wanneer die zijn geconfigureerd; anders gebruikt de ACP-
  harness zijn eigen standaardmodel. Andere harnesses moeten ACP
  `models` bekendmaken en `session/set_model` ondersteunen; anders mislukt OpenClaw/acpx
  met een duidelijke fout in plaats van stilzwijgend terug te vallen op de standaardwaarde van de doelagent.
</ParamField>
<ParamField path="thinking" type="string">
  Expliciete denk-/redeneerinspanning. Voor Codex ACP wordt `minimal` toegewezen aan een lage
  inspanning, worden `low`/`medium`/`high`/`xhigh` rechtstreeks toegewezen en laat `off` de
  overschrijving van de redeneerinspanning bij het opstarten weg. Wanneer dit wordt weggelaten, gebruiken ACP-starts bestaande
  standaardwaarden voor het denken van subagents en de modelspecifieke
  `agents.defaults.models["provider/model"].params.thinking` voor het geselecteerde
  model.
</ParamField>

## Koppelings- en threadmodi voor starten

<Tabs>
  <Tab title="--bind here|off">
    | Modus   | Gedrag                                                               |
    | ------ | ----------------------------------------------------------------------- |
    | `here` | Koppel het huidige actieve gesprek ter plaatse; mislukt als er geen gesprek actief is. |
    | `off`  | Maak geen koppeling met het huidige gesprek.                          |

    Opmerkingen:

    - `--bind here` is voor de operator het eenvoudigste pad om „dit kanaal of deze chat door Codex te laten ondersteunen”.
    - `--bind here` maakt geen onderliggende thread.
    - `--bind here` is alleen beschikbaar op kanalen die ondersteuning bieden voor koppeling met het huidige gesprek.
    - `--bind` en `--thread` kunnen niet in dezelfde aanroep van `/acp spawn` worden gecombineerd.

  </Tab>
  <Tab title="--thread auto|here|off">
    | Modus   | Gedrag                                                                                            |
    | ------ | ------------------------------------------------------------------------------------------------- |
    | `auto` | In een actieve thread: koppel die thread. Buiten een thread: maak en koppel waar ondersteund een onderliggende thread. |
    | `here` | Vereis een momenteel actieve thread; mislukt als je je niet in een thread bevindt.                                                  |
    | `off`  | Geen koppeling. De sessie start ongekoppeld.                                                                 |

    Opmerkingen:

    - Op oppervlakken zonder threadkoppeling is het standaardgedrag feitelijk `off`.
    - Voor een aan een thread gekoppelde start is ondersteuning door het kanaalbeleid vereist:
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`
    - Gebruik `--bind here` wanneer je het huidige gesprek wilt vastzetten zonder een onderliggende thread te maken.

  </Tab>
</Tabs>

## Leveringsmodel

ACP-sessies kunnen interactieve werkruimten of achtergrondwerk in beheer van de
bovenliggende sessie zijn. Het leveringspad hangt van die vorm af.

<AccordionGroup>
  <Accordion title="Interactieve ACP-sessies">
    Interactieve sessies zijn bedoeld om op een zichtbaar chatoppervlak te blijven communiceren:

    - `/acp spawn ... --bind here` koppelt het huidige gesprek aan de ACP-sessie.
    - `/acp spawn ... --thread ...` koppelt een kanaalthread/-onderwerp aan de ACP-sessie.
    - Permanente, geconfigureerde `bindings[].type="acp"` routeren overeenkomende gesprekken naar dezelfde ACP-sessie.

    Vervolgberichten in het gekoppelde gesprek worden rechtstreeks naar de ACP-
    sessie gerouteerd en ACP-uitvoer wordt teruggeleverd aan hetzelfde
    kanaal/dezelfde thread/hetzelfde onderwerp.

    Wat OpenClaw naar de harness verzendt:

    - Normale gebonden vervolgberichten worden als prompttekst verzonden, plus bijlagen, maar alleen wanneer de harness/backend die ondersteunt.
    - `/acp`-beheeropdrachten en lokale Gateway-opdrachten worden vóór ACP-dispatch onderschept.
    - Tijdens runtime gegenereerde voltooiingsgebeurtenissen worden per doel gematerialiseerd. OpenClaw-agenten krijgen de interne runtime-contextenvelop van OpenClaw; externe ACP-harnesses krijgen een gewone prompt met het resultaat van het kind en de instructie. De onbewerkte `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>`-envelop mag nooit naar externe harnesses worden verzonden of als ACP-gebruikerstranscripttekst worden opgeslagen.
    - ACP-transcriptvermeldingen gebruiken de voor de gebruiker zichtbare triggertekst of de gewone voltooiingsprompt. Interne gebeurtenismetadata blijven waar mogelijk gestructureerd in OpenClaw en worden niet behandeld als door de gebruiker geschreven chatinhoud.

  </Accordion>
  <Accordion title="Eenmalige ACP-sessies in beheer van de ouder">
    Eenmalige ACP-sessies die door een andere agentrun worden gestart, zijn
    achtergrondkinderen, vergelijkbaar met subagenten:

    - De ouder vraagt om werk met `sessions_spawn({ runtime: "acp", mode: "run" })`.
    - Het kind wordt uitgevoerd in een eigen ACP-harnesssessie.
    - Kindbeurten worden uitgevoerd in dezelfde achtergrondbaan als die voor het starten van native subagenten, zodat een trage ACP-harness niet-gerelateerd werk in de hoofdsessie niet blokkeert.
    - De voltooiing wordt teruggemeld via het aankondigingspad voor taakvoltooiing. OpenClaw zet interne voltooiingsmetadata om in een gewone ACP-prompt voordat die naar een externe harness wordt verzonden, zodat harnesses geen runtime-contextmarkeringen zien die alleen voor OpenClaw bestemd zijn.
    - De ouder herschrijft het resultaat van het kind in een normale assistentstem wanneer een gebruikersgericht antwoord nuttig is.

    Behandel dit pad **niet** als een peer-to-peerchat tussen ouder en
    kind. Het kind heeft al een voltooiingskanaal terug naar de ouder.

  </Accordion>
  <Accordion title="sessions_send en A2A-bezorging">
    `sessions_send` kan na het starten een andere sessie als doel kiezen. Voor normale
    peersessies gebruikt OpenClaw een agent-naar-agentvervolgpad (A2A) nadat
    het bericht is geïnjecteerd:

    - Wacht op het antwoord van de doelsessie.
    - Laat de aanvrager en het doel eventueel een begrensd aantal vervolgbeurten uitwisselen.
    - Vraag het doel om een aankondigingsbericht te produceren.
    - Bezorg die aankondiging in het zichtbare kanaal of de zichtbare thread.

    Dat A2A-pad is een terugvaloptie voor verzendingen naar peers waarbij de
    afzender een zichtbaar vervolgbericht nodig heeft. Het blijft ingeschakeld
    wanneer een niet-gerelateerde sessie een ACP-doel kan zien en berichten
    kan sturen, bijvoorbeeld bij ruime `tools.sessions.visibility`-instellingen.

    OpenClaw slaat het A2A-vervolg alleen over wanneer de aanvrager de ouder
    is van zijn eigen eenmalige ACP-kind dat door de ouder wordt beheerd. In
    dat geval kan A2A boven op taakvoltooiing de ouder wekken met het resultaat
    van het kind, het antwoord van de ouder terugsturen naar het kind en een
    echolus tussen ouder en kind veroorzaken. Het resultaat van
    `sessions_send` meldt voor dat beheerde kindgeval `delivery.status="skipped"`,
    omdat het voltooiingspad al verantwoordelijk is voor het resultaat.

  </Accordion>
  <Accordion title="Een bestaande sessie hervatten">
    Gebruik `resumeSessionId` om een eerdere ACP-sessie voort te zetten in
    plaats van opnieuw te beginnen. De agent speelt de gespreksgeschiedenis
    opnieuw af via `session/load`, zodat deze verdergaat met de volledige
    context van wat eraan voorafging.

    ```json
    {
      "task": "Ga verder waar we gebleven waren - los de resterende testfouten op",
      "runtime": "acp",
      "agentId": "codex",
      "resumeSessionId": "<previous-session-id>"
    }
    ```

    Veelvoorkomende gebruiksscenario's:

    - Draag een Codex-sessie over van je laptop naar je telefoon: vertel je agent dat deze verder moet gaan waar je gebleven was.
    - Zet een codeersessie die je interactief in de CLI hebt gestart nu zonder gebruikersinterface voort via je agent.
    - Hervat werk dat door een herstart van de Gateway of een time-out wegens inactiviteit is onderbroken.

    Opmerkingen:

    - `resumeSessionId` is alleen van toepassing wanneer `runtime: "acp"`; de standaardruntime voor subagenten negeert dit veld dat alleen voor ACP bestemd is.
    - `streamTo` is alleen van toepassing wanneer `runtime: "acp"`; de standaardruntime voor subagenten negeert dit veld dat alleen voor ACP bestemd is.
    - `resumeSessionId` is een hostlokale hervattings-id voor ACP/harnesses, geen OpenClaw-kanaalsessiesleutel; OpenClaw controleert vóór dispatch nog steeds het ACP-startbeleid en het beleid van de doelagent, terwijl de ACP-backend of harness verantwoordelijk is voor autorisatie om die upstream-id te laden.
    - `resumeSessionId` herstelt de upstream-ACP-gespreksgeschiedenis; `thread` en `mode` zijn nog steeds normaal van toepassing op de nieuwe OpenClaw-sessie die je maakt, dus `mode: "session"` vereist nog steeds `thread: true`.
    - De doelagent moet `session/load` ondersteunen (Codex en Claude Code doen dat).
    - Als de sessie-id niet wordt gevonden, mislukt het starten met een duidelijke foutmelding; er is geen stille terugval naar een nieuwe sessie.

  </Accordion>
  <Accordion title="Smoketest na implementatie">
    Voer na een Gateway-implementatie een live end-to-endcontrole uit in plaats
    van op unittests te vertrouwen:

    1. Controleer de geïmplementeerde Gateway-versie en commit op de doelhost.
    2. Open een tijdelijke ACPX-brugsessie naar een live agent.
    3. Vraag die agent om `sessions_spawn` aan te roepen met `runtime: "acp"`, `agentId: "codex"`, `mode: "run"` en taak `Reply with exactly LIVE-ACP-SPAWN-OK`.
    4. Controleer `accepted=yes`, een echte `childSessionKey` en dat er geen validatiefout is.
    5. Ruim de tijdelijke brugsessie op.

    Houd de gate op `mode: "run"` en sla `streamTo: "parent"` over:
    threadgebonden `mode: "session"`- en streamrelaypaden zijn afzonderlijke,
    uitgebreidere integratiecontroles.

  </Accordion>
</AccordionGroup>

## Compatibiliteit met de sandbox

ACP-sessies worden momenteel uitgevoerd op de hostruntime, **niet** in de
OpenClaw-sandbox.

<Warning>
**Beveiligingsgrens:**

- De externe harness kan lezen/schrijven volgens zijn eigen CLI-machtigingen en de geselecteerde `cwd`.
- Het sandboxbeleid van OpenClaw omvat de uitvoering van ACP-harnesses **niet**.
- OpenClaw handhaaft nog steeds ACP-featuregates, toegestane agenten, sessie-eigendom, kanaalbindingen en het bezorgingsbeleid van de Gateway.
- Gebruik `runtime: "subagent"` voor OpenClaw-native werk waarbij de sandbox wordt gehandhaafd.

</Warning>

Huidige beperkingen:

- Als de aanvragersessie in een sandbox wordt uitgevoerd, wordt het starten van ACP geblokkeerd voor zowel `sessions_spawn({ runtime: "acp" })` als `/acp spawn`.
- `sessions_spawn` met `runtime: "acp"` ondersteunt `sandbox: "require"` niet.

## Doelsessies bepalen

De meeste `/acp`-acties accepteren een optioneel sessiedoel (`session-key`,
`session-id` of `session-label`).

**Volgorde voor het bepalen:**

1. Expliciet doelargument (of `--session` voor `/acp steer`)
   - probeert eerst de sleutel
   - daarna een UUID-vormige sessie-id
   - daarna het label
2. Huidige threadbinding (als dit gesprek/deze thread aan een ACP-sessie is gebonden).
3. Terugval op de huidige aanvragersessie.

Zowel bindingen van het huidige gesprek als threadbindingen doen mee aan stap 2.

Als er geen doel kan worden bepaald, retourneert OpenClaw een duidelijke fout
(`Unable to resolve session target: ...`).

## ACP-bediening

| Opdracht              | Functie                                              | Voorbeeld                                                       |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | Maak een ACP-sessie; optioneel met huidige binding of threadbinding. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Annuleer de lopende beurt voor de doelsessie.                 | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Stuur een bijsturingsinstructie naar de actieve sessie.                | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Sluit de sessie en ontkoppel threaddoelen.                  | `/acp close`                                                  |
| `/acp status`        | Toon backend, modus, status, runtimeopties en mogelijkheden. | `/acp status`                                                 |
| `/acp set-mode`      | Stel de runtimemodus voor de doelsessie in.                      | `/acp set-mode plan`                                          |
| `/acp set`           | Schrijf een algemene runtimeconfiguratieoptie.                      | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Stel een vervangende runtimewerkmap in.                   | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Stel het goedkeuringsbeleidsprofiel in.                              | `/acp permissions strict`                                     |
| `/acp timeout`       | Stel de runtime-time-out in (seconden).                            | `/acp timeout 120`                                            |
| `/acp model`         | Stel een vervangend runtimemodel in.                               | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Verwijder vervangende runtimeopties voor de sessie.                  | `/acp reset-options`                                          |
| `/acp sessions`      | Toon recente ACP-sessies uit de opslag.                      | `/acp sessions`                                               |
| `/acp doctor`        | Backendstatus, mogelijkheden en uitvoerbare oplossingen.           | `/acp doctor`                                                 |
| `/acp install`       | Druk deterministische installatie- en activeringsstappen af.             | `/acp install`                                                |

Runtimebediening (`spawn`, `cancel`, `steer`, `close`, `status`, `set-mode`,
`set`, `cwd`, `permissions`, `timeout`, `model` en `reset-options`) vereist
een eigenaarsidentiteit van externe kanalen en `operator.admin` van interne
Gateway-clients. Geautoriseerde afzenders die geen eigenaar zijn, kunnen nog steeds `sessions`,
`doctor`, `install` en `help` gebruiken. Voor afzenders die geen eigenaar zijn, toont `/acp sessions`
alleen de huidige gebonden sessie of aanvragersessie; eigenaarsidentiteiten en
`operator.admin`-clients zien alle recente sessies.

`/acp status` toont de effectieve runtimeopties plus sessie-id's op
runtime- en backendniveau. Fouten voor niet-ondersteunde bediening worden
duidelijk weergegeven wanneer een backend een mogelijkheid mist. Opdrachten die
doeltokens accepteren (`session-key`, `session-id` of `session-label`) bepalen deze via Gateway-
sessiedetectie, inclusief aangepaste `session.store`-roots per agent. `/acp sessions`
accepteert geen doeltoken.

### Toewijzing van runtimeopties

`/acp` heeft gemaksopdrachten en een algemene setter. Gelijkwaardige bewerkingen:

| Opdracht                      | Komt overeen met                              | Opmerkingen                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/acp model <id>`            | runtimeconfiguratiesleutel `model`           | Voor Codex ACP normaliseert OpenClaw `openai/<model>` naar de model-id van de adapter en zet het suffix voor redeneerniveau na een slash, zoals `openai/gpt-5.4/high`, om naar `reasoning_effort`.                                         |
| `/acp set thinking <level>`  | canonieke optie `thinking`          | OpenClaw verzendt het door de backend geadverteerde equivalent wanneer dit beschikbaar is, met voorkeur voor `thinking`, gevolgd door `effort`, `reasoning_effort` of `thought_level`. Voor Codex ACP zet de adapter waarden om naar `reasoning_effort`. |
| `/acp permissions <profile>` | canonieke optie `permissionProfile` | OpenClaw verzendt het door de backend geadverteerde equivalent wanneer dit beschikbaar is, zoals `approval_policy`, `permission_profile`, `permissions` of `permission_mode`.                                                       |
| `/acp timeout <seconds>`     | canonieke optie `timeoutSeconds`    | OpenClaw verzendt het door de backend geadverteerde equivalent wanneer dit beschikbaar is, zoals `timeout` of `timeout_seconds`.                                                                                                     |
| `/acp cwd <path>`            | overschrijving van runtimewerkmap                 | Rechtstreekse update.                                                                                                                                                                                             |
| `/acp set <key> <value>`     | algemeen                              | `key=cwd` gebruikt het overschreven werkmappad.                                                                                                                                                                      |
| `/acp reset-options`         | wist alle runtimeoverschrijvingen         | -                                                                                                                                                                                                          |

## acpx-harnas, Plugin-installatie en machtigingen

Zie [ACP-agents - installatie](/nl/tools/acp-agents-setup) voor de configuratie van het acpx-harnas
(aliassen voor Claude Code / Codex / Gemini CLI), de MCP-bruggen voor
Plugin-tools en OpenClaw-tools, en de ACP-machtigingsmodi.

## Problemen oplossen

| Symptoom                                                                                   | Waarschijnlijke oorzaak                                                                                                           | Oplossing                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACP runtime backend is not configured`                                                   | Backend-Plugin ontbreekt, is uitgeschakeld of wordt geblokkeerd door `plugins.allow`.                                                       | Installeer en activeer de backend-Plugin, neem `acpx` op in `plugins.allow` wanneer die toelatingslijst is ingesteld en voer vervolgens `/acp doctor` uit.                                                 |
| `ACP is disabled by policy (acp.enabled=false)`                                           | ACP is globaal uitgeschakeld.                                                                                                 | Stel `acp.enabled=true` in.                                                                                                                                                  |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`                         | Automatische verzending vanuit normale threadberichten is uitgeschakeld.                                                               | Stel `acp.dispatch.enabled=true` in om automatische threadroutering te hervatten; expliciete aanroepen van `sessions_spawn({ runtime: "acp" })` blijven werken.                                      |
| `ACP agent "<id>" is not allowed by policy`                                               | Agent staat niet op de toelatingslijst.                                                                                                | Gebruik een toegestane `agentId` of werk `acp.allowedAgents` bij.                                                                                                                     |
| `/acp doctor` meldt direct na het opstarten dat de backend niet gereed is                               | De backend-Plugin ontbreekt, is uitgeschakeld, wordt geblokkeerd door het toelatings-/weigeringsbeleid of het geconfigureerde uitvoerbare bestand is niet beschikbaar.        | Installeer/activeer de backend-Plugin, voer `/acp doctor` opnieuw uit en controleer de installatiefout of beleidsfout van de backend als deze ongezond blijft.                                           |
| Harnasopdracht niet gevonden                                                                 | De adapter-CLI is niet geïnstalleerd, de externe Plugin ontbreekt of het ophalen via `npx` bij de eerste uitvoering is mislukt voor een niet-Codex-adapter. | Voer `/acp doctor` uit, installeer/verwarm de adapter vooraf op de Gateway-host of configureer de opdracht voor de acpx-agent expliciet.                                                      |
| Model niet gevonden door het harnas                                                          | De model-id is geldig voor een andere provider/ander harnas, maar niet voor dit ACP-doel.                                                | Gebruik een model dat door dat harnas wordt vermeld, configureer het model in het harnas of laat de overschrijving weg.                                                                            |
| Leveranciersauthenticatiefout van het harnas                                                        | OpenClaw functioneert, maar er is niet ingelogd bij de doel-CLI/provider.                                                     | Log in of geef de vereiste providersleutel op in de omgeving van de Gateway-host.                                                                                             |
| `Unable to resolve session target: ...`                                                   | Ongeldig sleutel-/id-/labeltoken.                                                                                                | Voer `/acp sessions` uit, kopieer de exacte sleutel/het exacte label en probeer het opnieuw.                                                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation`               | `--bind here` gebruikt zonder een actief koppelbaar gesprek.                                                            | Ga naar de doelchat/het doelkanaal en probeer het opnieuw, of start zonder koppeling.                                                                                                         |
| `Conversation bindings are unavailable for <channel>.`                                    | De adapter kan ACP niet aan het huidige gesprek koppelen.                                                             | Gebruik `/acp spawn ... --thread ...` waar dit wordt ondersteund, configureer `bindings[]` op het hoogste niveau of ga naar een ondersteund kanaal.                                                     |
| `--thread here requires running /acp spawn inside an active ... thread`                   | `--thread here` gebruikt buiten een threadcontext.                                                                         | Ga naar de doelthread of gebruik `--thread auto`/`off`.                                                                                                                      |
| `Only <user-id> can rebind this channel/conversation/thread.`                             | Een andere gebruiker is eigenaar van het actieve koppelingsdoel.                                                                           | Koppel opnieuw als eigenaar of gebruik een ander gesprek of een andere thread.                                                                                                               |
| `Thread bindings are unavailable for <channel>.`                                          | De adapter kan geen threads koppelen.                                                                               | Gebruik `--thread off` of ga naar een ondersteunde adapter/ondersteund kanaal.                                                                                                                 |
| `Sandboxed sessions cannot spawn ACP sessions ...`                                        | De ACP-runtime draait aan de hostzijde; de sessie van de aanvrager bevindt zich in een sandbox.                                                              | Gebruik `runtime="subagent"` vanuit sessies in een sandbox of start ACP vanuit een sessie die zich niet in een sandbox bevindt.                                                                         |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`                   | `sandbox="require"` aangevraagd voor de ACP-runtime.                                                                         | Gebruik `runtime="subagent"` als sandboxing vereist is, of gebruik ACP met `sandbox="inherit"` vanuit een sessie die zich niet in een sandbox bevindt.                                                      |
| `Cannot apply --model ... did not advertise model support`                                | Het doelharnas biedt geen algemene ACP-modelwisseling.                                                        | Gebruik een harnas dat ACP `models`/`session/set_model` adverteert, gebruik Codex ACP-modelreferenties of configureer het model rechtstreeks in het harnas als dit een eigen opstartvlag heeft. |
| Ontbrekende ACP-metadata voor gekoppelde sessie                                                    | Verouderde/verwijderde ACP-sessiemetadata.                                                                                    | Maak deze opnieuw aan met `/acp spawn` en koppel/focus daarna de thread opnieuw.                                                                                                                    |
| `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` | `permissionMode` blokkeert schrijfbewerkingen/uitvoering in een niet-interactieve ACP-sessie.                                                    | Stel `plugins.entries.acpx.config.permissionMode` in op `approve-all` en start de Gateway opnieuw. Zie [Machtigingsconfiguratie](/nl/tools/acp-agents-setup#permission-configuration). |
| ACP-sessie mislukt vroegtijdig met weinig uitvoer                                                | Machtigingsprompts worden geblokkeerd door `permissionMode`/`nonInteractivePermissions`.                                        | Controleer de Gateway-logboeken op `AcpRuntimeError`. Stel voor volledige machtigingen `permissionMode=approve-all` in; stel voor geleidelijke degradatie `nonInteractivePermissions=deny` in.        |
| ACP-sessie blijft na voltooiing van het werk onbeperkt hangen                                     | Het harnasproces is voltooid, maar de ACP-sessie heeft geen voltooiing gemeld.                                                    | Werk OpenClaw bij; de huidige acpx-opruiming beëindigt verouderde wrapper- en adapterprocessen die eigendom zijn van OpenClaw bij het sluiten en bij het opstarten van de Gateway.                                             |
| Harnas ziet `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>`                                      | Interne gebeurtenisenvelop is over de ACP-grens gelekt.                                                                | Werk OpenClaw bij en voer de voltooiingsflow opnieuw uit; externe harnassen horen alleen gewone voltooiingsprompts te ontvangen.                                                          |

<Note>
`Command blocked by PreToolUse hook: Native hook relay unavailable` hoort bij
de systeemeigen Codex-hookrelay, niet bij ACP/acpx. Start in een gekoppelde Codex-chat een
nieuwe sessie met `/new` of `/reset`; als dit eenmaal werkt en vervolgens bij
de volgende systeemeigen toolaanroep terugkeert, start je de Codex-appserver of OpenClaw Gateway
opnieuw in plaats van `/new` te herhalen. Zie
[Problemen met het Codex-harnas oplossen](/nl/plugins/codex-harness#troubleshooting).
</Note>

## Gerelateerd

- [ACP-agents - configuratie](/nl/tools/acp-agents-setup)
- [Agent verzenden](/nl/tools/agent-send)
- [CLI-backends](/nl/gateway/cli-backends)
- [Codex-harnas](/nl/plugins/codex-harness)
- [Codex-harnasruntime](/nl/plugins/codex-harness-runtime)
- [Multi-agent-sandboxtools](/nl/tools/multi-agent-sandbox-tools)
- [`openclaw acp` (brugmodus)](/nl/cli/acp)
- [Subagents](/nl/tools/subagents)
