---
doc-schema-version: 1
read_when:
    - Je wilt begrijpen welke tools OpenClaw biedt
    - Je kiest tussen ingebouwde tools, Skills en plugins
    - Je hebt het juiste startpunt in de documentatie nodig voor toolbeleid, automatisering of agentcoördinatie
summary: 'Overzicht van OpenClaw-tools, Skills en plugins: wat agents kunnen aanroepen en hoe je ze kunt uitbreiden'
title: Overzicht
x-i18n:
    generated_at: "2026-07-27T05:26:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45745bd5f2008a84cb6c4c1c9840073bfa8a9c40a0ff65bfefc682c5d99af09b
    source_path: tools/index.md
    workflow: 16
---

Gebruik deze pagina om het juiste oppervlak voor mogelijkheden te kiezen. **Tools** zijn
aanroepbare acties, **Skills** leren agents hoe ze moeten werken en **plugins** voegen
runtimemogelijkheden toe, zoals tools, providers, kanalen, hooks en gebundelde
Skills.

Dit is een overzichts- en routeringspagina. Raadpleeg voor volledig toolbeleid, standaardinstellingen,
groepslidmaatschap, providerbeperkingen en configuratievelden
[Tools en aangepaste providers](/nl/gateway/config-tools).

## Begin hier

Begin voor de meeste agents met de ingebouwde toolcategorieën en pas daarna het beleid
alleen aan wanneer de agent minder tools mag zien of expliciete hosttoegang nodig heeft.

| Als je het volgende wilt...                              | Gebruik eerst dit                                  | Lees daarna                                                                                                                                                       |
| ------------------------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Een agent laten handelen met bestaande mogelijkheden    | [Ingebouwde tools](#built-in-tool-categories)      | [Toolcategorieën](#built-in-tool-categories)                                                                                                                      |
| Bepalen wat een agent kan aanroepen                     | [Toolbeleid](#configure-access-and-approvals)      | [Tools en aangepaste providers](/nl/gateway/config-tools)                                                                                                            |
| Een agent een workflow leren                            | [Skills](#choose-tools-skills-or-plugins)          | [Skills](/nl/tools/skills), [Skills maken](/nl/tools/creating-skills), [Skill Workshop](/nl/tools/skill-workshop) en [Zelflerend vermogen](/nl/tools/self-learning)            |
| Een nieuwe integratie of nieuw runtimeoppervlak toevoegen | [Plugins](#extend-capabilities)                  | [Plugins](/nl/tools/plugin) en [Plugins bouwen](/nl/plugins/building-plugins)                                                                                            |
| Werk later of op de achtergrond uitvoeren              | [Automatisering](/nl/automation)                      | [Overzicht van automatisering](/nl/automation)                                                                                                                       |
| Meerdere agents of harnassen coördineren                | [Subagents](/nl/tools/subagents)                      | [ACP-agents](/nl/tools/acp-agents) en [Agent verzenden](/nl/tools/agent-send)                                                                                            |
| Gelijktijdige agents vanuit code orkestreren            | [Zwerm](/tools/swarm)                              | [Codemodus](/nl/tools/code-mode) en [Subagents](/nl/tools/subagents)                                                                                                    |
| Een grote OpenClaw-toolcatalogus doorzoeken             | [Tools zoeken](/nl/tools/tool-search)                 | [Tools zoeken](/nl/tools/tool-search)                                                                                                                                |
| Meerdere tools combineren in één compact programma      | [Codemodus](/nl/tools/code-mode)                      | [Codemodus](/nl/tools/code-mode)                                                                                                                                     |

## Kies tools, Skills of plugins

<Steps>
  <Step title="Gebruik een tool wanneer de agent moet handelen">
    Een tool is een getypeerde functie die de agent kan aanroepen, zoals `exec`, `browser`,
    `web_search`, `message` of `image_generate`. Gebruik tools wanneer de agent
    gegevens moet lezen, bestanden moet wijzigen, berichten moet verzenden, een provider moet aanroepen of
    een ander systeem moet bedienen. Zichtbare tools worden als gestructureerde
    functiedefinities naar het model verzonden.

    Het model ziet alleen tools die overblijven na toepassing van het actieve profiel, het toestaan/weigeren-beleid,
    providerbeperkingen, de sandboxstatus, kanaalmachtigingen en
    de beschikbaarheid van plugins.

  </Step>

  <Step title="Gebruik een Skill wanneer de agent instructies nodig heeft">
    Een Skill is een `SKILL.md`-instructiepakket dat in de prompt van de agent wordt geladen. Gebruik
    een Skill wanneer de agent al over de benodigde tools beschikt, maar een
    herhaalbare workflow, beoordelingsrubriek, opdrachtenreeks of operationele
    beperking nodig heeft.

    Skills kunnen zich bevinden in een werkruimte, een gedeelde Skills-map, de beheerde hoofdmap voor
    OpenClaw-Skills of een pluginpakket.

    [Skills](/nl/tools/skills) | [Skill Workshop](/nl/tools/skill-workshop) | [Zelflerend vermogen](/nl/tools/self-learning) | [Skills maken](/nl/tools/creating-skills) | [Skills-configuratie](/nl/tools/skills-config)

  </Step>

  <Step title="Gebruik een plugin wanneer OpenClaw een nieuwe mogelijkheid nodig heeft">
    Een plugin kan tools, Skills, kanalen, modelproviders, spraak,
    realtime spraak, mediageneratie, zoeken op het web, webinhoud ophalen, hooks en andere
    runtimemogelijkheden toevoegen. Gebruik een plugin wanneer de mogelijkheid code,
    aanmeldgegevens, levenscyclushooks, manifestmetagegevens of installeerbare
    pakketten omvat. Bestaande plugins kunnen worden geïnstalleerd vanuit ClawHub, npm, git,
    lokale mappen of archieven.

    [Plugins installeren en configureren](/nl/tools/plugin) | [Plugins bouwen](/nl/plugins/building-plugins) | [Plugin-SDK](/nl/plugins/sdk-overview)

  </Step>
</Steps>

## Ingebouwde toolcategorieën

De tabel vermeldt representatieve tools, zodat je het oppervlak kunt herkennen. Dit is
niet de volledige beleidsreferentie. Raadpleeg voor de exacte groepen, standaardinstellingen en semantiek voor
toestaan/weigeren [Tools en aangepaste providers](/nl/gateway/config-tools).

| Categorie                | Gebruik wanneer de agent het volgende moet...                                                   | Representatieve tools                                                                                                | Lees daarna                                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Runtime                  | Opdrachten uitvoeren, processen beheren of door providers ondersteunde Python-analyse gebruiken | `exec`, `process`, `terminal`, `code_execution`                                     | [Exec](/nl/tools/exec), [Control UI-terminal](/nl/web/control-ui#operator-terminal), [Code-uitvoering](/nl/tools/code-execution)           |
| Bestanden                | Werkruimtebestanden lezen en wijzigen                                                            | `read`, `write`, `edit`, `apply_patch`                                     | [Patch toepassen](/nl/tools/apply-patch)                                                                                             |
| Menselijke invoer        | Pauzeren voor een gestructureerde beslissing die bij de gebruiker ligt                           | `ask_user`                                                                                                  | [Gebruiker vragen](/tools/ask-user)                                                                                               |
| Web                      | Het web of berichten op X doorzoeken, of leesbare pagina-inhoud ophalen                           | `web_search`, `x_search`, `web_fetch`                                                         | [Webtools](/nl/tools/web), [Webinhoud ophalen](/nl/tools/web-fetch)                                                                     |
| Browser                  | Een browsersessie bedienen                                                                       | `browser`                                                                                                  | [Browser](/nl/tools/browser)                                                                                                         |
| Operatorinterface        | Verbonden deelvensters, panelen en navigatie van de Control UI ordenen                            | `screen`                                                                                                  | [Scherm](/tools/screen)                                                                                                           |
| Berichten en kanalen     | Antwoorden verzenden of kanaalacties uitvoeren                                                    | `message`                                                                                                  | [Agent verzenden](/nl/tools/agent-send)                                                                                              |
| Sessies en agents        | Sessies inspecteren, werk delegeren, collectors orkestreren, een andere uitvoering aansturen of status rapporteren | `sessions_*`, `agents_wait`, `subagents`, `agents_list`, `session_status`, `get_goal`, `create_goal`, `update_goal` | [Doel](/nl/tools/goal), [Zwerm](/tools/swarm), [Subagents](/nl/tools/subagents), [Sessietool](/nl/concepts/session-tool)                    |
| Automatisering           | Werk plannen of reageren op achtergrondgebeurtenissen                                            | `cron`, `heartbeat_respond`                                                                              | [Automatisering](/nl/automation)                                                                                                     |
| Gateway en nodes         | De status van de Gateway of gekoppelde doelapparaten inspecteren                                  | `gateway`, `nodes`                                                                              | [Gateway-configuratie](/nl/gateway/configuration), [Nodes](/nl/nodes)                                                                   |
| Media                    | Media analyseren, genereren of uitspreken                                                         | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                 | [Mediaoverzicht](/nl/tools/media-overview)                                                                                           |
| Grote OpenClaw-catalogi  | Veel geschikte tools zoeken, aanroepen en combineren zonder elk schema naar het model te verzenden | `exec`, `wait`, `tool_search_code`, `tool_search`, `tool_describe`                 | [Codemodus](/nl/tools/code-mode), [Tools zoeken](/nl/tools/tool-search)                                                                 |

<Note>
Codemodus en Tools zoeken zijn experimentele OpenClaw-agentoppervlakken. Uitvoeringen met het Codex-
harnas gebruiken de Codex-eigen codemodus, systeemeigen toolzoekfunctie, uitgestelde dynamische
tools en geneste toolaanroepen in plaats van `tools.codeMode` of `tools.toolSearch`.
</Note>

## Door plugins geleverde tools

Plugins kunnen aanvullende tools registreren. Pluginauteurs koppelen tools via
`api.registerTool(...)` en `contracts.tools` van het manifest; raadpleeg
[Plugin-SDK](/nl/plugins/sdk-overview) en [Pluginmanifest](/nl/plugins/manifest)
voor details over het contract.

Veelvoorkomende door plugins geleverde tools zijn:

- [Diffs](/nl/tools/diffs) voor het weergeven van bestands- en Markdown-diffs
- [Widget weergeven](/nl/tools/show-widget) voor zelfstandige inline-SVG en -HTML in ondersteunde chatclients
- [Scherm](/tools/screen) voor het indelen van een verbonden Control UI
- [LLM-taak](/nl/tools/llm-task) voor workflowstappen met uitsluitend JSON
- [Lobster](/nl/tools/lobster) voor getypeerde workflows met hervatbare goedkeuringen
- [Tokenjuice](/nl/tools/tokenjuice) voor het compact maken van onoverzichtelijke uitvoer van de tools
  `exec` en `bash`
- [Tools zoeken](/nl/tools/tool-search) voor het vinden en aanroepen van grote
  toolcatalogi zonder elk schema in de prompt op te nemen
- [Canvas](/nl/plugins/reference/canvas) voor Node Canvas-besturing en A2UI-
  weergave

## Toegang en goedkeuringen configureren

Het toolbeleid wordt vóór de modelaanroep afgedwongen. Als het beleid een tool verwijdert,
ontvangt het model het schema van die tool niet voor die beurt. Een uitvoering kan tools
kwijtraken door globale configuratie, configuratie per agent, kanaalbeleid, beperkingen
van de provider, sandboxregels, kanaal-/runtimebeleid of beschikbaarheid van plugins.

- [Tools en aangepaste providers](/nl/gateway/config-tools) documenteert toolprofielen,
  lijsten met toegestane/geweigerde items, providerspecifieke beperkingen, lusdetectie en
  instellingen voor door providers ondersteunde tools.
- [Exec-goedkeuringen](/nl/tools/exec-approvals) documenteert het goedkeuringsbeleid
  voor hostopdrachten.
- [Verhoogde exec](/nl/tools/elevated) documenteert gecontroleerde uitvoering buiten de
  sandbox.
- [Sandbox versus toolbeleid versus verhoogde toegang](/nl/gateway/sandbox-vs-tool-policy-vs-elevated)
  legt uit welke laag de toegang tot bestanden en processen beheert.
- [Sandbox- en toolbeperkingen per agent](/nl/tools/multi-agent-sandbox-tools)
  documenteert agentspecifieke beperkingen voor gedelegeerde uitvoeringen.

## Mogelijkheden uitbreiden

Kies het uitbreidingspad op basis van de taak die OpenClaw moet uitvoeren:

- Installeer of beheer een bestaande plugin met [Plugins](/nl/tools/plugin).
- Bouw een nieuwe integratie, provider, kanaal, tool of hook met
  [Plugins bouwen](/nl/plugins/building-plugins).
- Voeg herbruikbare agentinstructies toe of stem ze af met [Skills](/nl/tools/skills) en
  [Skills maken](/nl/tools/creating-skills).
- Gebruik [Plugin SDK](/nl/plugins/sdk-overview) en
  [Pluginmanifest](/nl/plugins/manifest) wanneer je implementatiecontracten
  nodig hebt.

## Ontbrekende tools oplossen

Als het model een tool niet kan zien of aanroepen, begin dan met het effectieve beleid voor
de huidige beurt:

1. Controleer het actieve profiel, `tools.allow` en `tools.deny` in
   [Tools en aangepaste providers](/nl/gateway/config-tools).
2. Controleer providerspecifieke beperkingen in
   [Tools en aangepaste providers](/nl/gateway/config-tools) en bevestig dat de
   geselecteerde [modelprovider](/nl/concepts/model-providers) de vorm van de tool
   ondersteunt.
3. Controleer kanaalmachtigingen, de sandboxstatus en verhoogde toegang met
   [Sandbox versus toolbeleid versus verhoogde toegang](/nl/gateway/sandbox-vs-tool-policy-vs-elevated)
   en [Verhoogde exec](/nl/tools/elevated).
4. Controleer in
   [Plugins](/nl/tools/plugin) of de bijbehorende plugin is geïnstalleerd en ingeschakeld.
5. Controleer voor gedelegeerde uitvoeringen de beperkingen per agent in
   [Sandbox- en toolbeperkingen per agent](/nl/tools/multi-agent-sandbox-tools).
6. Bevestig voor grote OpenClaw-catalogi of de uitvoering directe beschikbaarstelling van tools,
   [Code Mode](/nl/tools/code-mode) of [Tools zoeken](/nl/tools/tool-search) gebruikt.

## Gerelateerd

- [Automatisering](/nl/automation) voor Cron, taken, Heartbeat, hooks,
  permanente opdrachten en Task Flow
- [Agents](/nl/concepts/agent) voor het agentmodel, sessies, geheugen en
  coördinatie tussen meerdere agents
- [Tools en aangepaste providers](/nl/gateway/config-tools) voor de canonieke referentie
  voor toolbeleid
- [Plugins](/nl/tools/plugin) voor installatie en beheer van plugins
- [Plugin SDK](/nl/plugins/sdk-overview) als referentie voor pluginauteurs
- [Skills](/nl/tools/skills) voor de laadvolgorde, gating en configuratie van skills
- [Skillworkshop](/nl/tools/skill-workshop) voor het genereren en beoordelen van
  skills
- [Tools zoeken](/nl/tools/tool-search) voor het compact ontdekken van de
  OpenClaw-toolcatalogus
- [Code Mode](/nl/tools/code-mode) voor compacte JavaScript- of TypeScript-workflows
  via een verborgen OpenClaw-toolcatalogus
- [Swarm](/tools/swarm) voor gestructureerde vertakking en verzameling vanuit Code Mode
