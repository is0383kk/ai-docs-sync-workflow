---
read_when:
    - Tekst van de systeemprompt, lijst met tools of tijd-/heartbeatsecties bewerken
    - Gedrag voor workspace-bootstrap of Skills-injectie wijzigen
summary: Wat de systeemprompt van OpenClaw bevat en hoe deze wordt samengesteld
title: Systeemprompt
x-i18n:
    generated_at: "2026-07-27T05:03:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 669fbc6f21a82a2c3c067d2ff3a6365acb3316460a85f2db165b7ad49ce79f70
    source_path: concepts/system-prompt.md
    workflow: 16
---

OpenClaw bouwt voor elke agentrun een eigen systeemprompt; er is geen standaardprompt tijdens runtime.

De samenstelling bestaat uit drie lagen:

- `buildAgentSystemPrompt` rendert de prompt vanuit expliciete invoer. Deze blijft een pure renderer en leest de globale configuratie niet rechtstreeks.
- `resolveAgentSystemPromptConfig` bepaalt voor een specifieke agent de configuratiegestuurde promptinstellingen (weergave van eigenaar, TTS-hints, modelaliassen, modus voor geheugencitaten, delegatiemodus voor subagents).
- Runtime-adapters (ingebed, CLI, opdracht-/exportvoorbeelden, Compaction) verzamelen actuele feiten (tools, sandboxstatus, kanaalmogelijkheden, contextbestanden, promptbijdragen van providers) en roepen de geconfigureerde promptfacade aan.

Hierdoor blijven geëxporteerde promptoppervlakken en foutopsporingsoppervlakken afgestemd op live-runs, zonder elk runtimedetail in één monolithische builder onder te brengen.

Providerplugins kunnen cachebewuste richtlijnen bijdragen zonder de prompt van OpenClaw te vervangen. Een providerruntime kan:

- een van drie benoemde kernsecties vervangen: `interaction_style`, `tool_call_style`, `execution_bias`
- een **stabiel voorvoegsel** boven de grens van de promptcache invoegen
- een **dynamisch achtervoegsel** onder de grens van de promptcache invoegen

Gebruik bijdragen van de provider voor modelspecifieke afstemming per modelfamilie. Reserveer de verouderde hook `before_prompt_build` voor compatibiliteit of werkelijk globale promptwijzigingen.

De meegeleverde OpenAI/Codex-overlay voor de GPT-5-familie (`resolveGpt5SystemPromptContribution`) gebruikt dit mechanisme: een `stablePrefix`-gedragscontract (uitvoeringsbeleid, tooldiscipline, uitvoercontract, voltooiingscontract) plus een optionele `interaction_style`-overschrijving voor een vriendelijkere toon. Deze is van toepassing op elke `gpt-5*`-model-id die via de OpenAI- of Codex-plugins wordt gerouteerd, aangestuurd door `agents.defaults.promptOverlays.gpt5.personality` (`"friendly"`/`"on"` of `"off"`).

## Structuur

De prompt is compact en heeft vaste secties:

- **Tooling**: herinnering aan gestructureerde tools als gezaghebbende bron, plus runtimerichtlijnen voor toolgebruik. Wanneer de experimentele tool `update_plan` is ingeschakeld (`tools.experimental.planTool`), voegt de eigen toolbeschrijving het volgende toe: gebruik deze alleen voor niet-triviaal werk met meerdere stappen, houd maximaal één stap `in_progress` en sla deze over voor eenvoudig werk met één stap.
- **Uitvoeringsvoorkeur**: handel tijdens de beurt naar uitvoerbare verzoeken, ga door tot het werk voltooid of geblokkeerd is, herstel van zwakke toolresultaten, controleer veranderlijke toestand live en verifieer vóór afronding.
- **Veiligheid**: korte herinnering aan waarborgen tegen machtszoekend gedrag of het omzeilen van toezicht.
- **Skills** (indien beschikbaar): vertelt het model hoe het instructies voor Skills naar behoefte kan laden.
- **OpenClaw-besturing**: geef de voorkeur aan de tool `gateway` voor configuratie- en herstartwerk; verzin geen CLI-opdrachten.
- **OpenClaw zelf bijwerken**: inspecteer de configuratie veilig met `config.schema.lookup`, pas deze aan met `config.patch`, vervang de volledige configuratie met `config.apply` en voer `update.run` alleen uit op uitdrukkelijk verzoek van de gebruiker. De agentgerichte tool `gateway` weigert `tools.exec.mode` te herschrijven.
- **Werkruimte**: werkmap (`agents.defaults.workspace`).
- **Documentatie**: lokaal pad naar documentatie/broncode en wanneer deze moet worden gelezen.
- **Werkruimtebestanden (ingevoegd)**: vermeldt dat bootstrapbestanden hieronder zijn opgenomen.
- **Sandbox** (wanneer ingeschakeld): runtime in sandbox, sandboxpaden, beschikbaarheid van uitvoering met verhoogde rechten.
- **Huidige datum en tijd**: alleen tijdzone (cachestabiel; de liveklok komt van `session_status`).
- **Richtlijnen voor assistentuitvoer**: compacte syntaxis voor bijlagen, spraakberichten en antwoordtags.
- **Heartbeats**: Heartbeat-prompt en bevestigingsgedrag, wanneer Heartbeats voor de standaardagent zijn ingeschakeld.
- **Runtime**: host, besturingssysteem, Node, model, hoofdmap van de repository (wanneer gedetecteerd), denkniveau (één regel).
- **Redenering**: huidig zichtbaarheidsniveau plus de hint voor de schakelaar `/reasoning`.

Grote stabiele inhoud (waaronder **Projectcontext**) blijft boven de interne grens van de promptcache. Veranderlijke secties per beurt (richtlijnen voor de ingesloten Control UI, **Berichten**, **Spraak**, **Groepschatcontext**, **Reacties**, **Heartbeats**, **Runtime**) worden onder die grens toegevoegd, zodat lokale backends met voorvoegselcaches het stabiele werkruimtevoorvoegsel opnieuw kunnen gebruiken tussen kanaalbeurten. Toolbeschrijvingen moeten vermijden huidige kanaalnamen op te nemen wanneer het geaccepteerde schema dat runtimedetail al bevat.

Tooling bevat ook richtlijnen voor langdurig werk:

- gebruik Cron voor toekomstige opvolging (`check back later`, herinneringen, terugkerend werk) in plaats van `exec`-slaaplussen, `yieldMs`-vertragingstrucs of herhaald pollen met `process`
- gebruik `exec` / `process` alleen voor opdrachten die nu starten en op de achtergrond doorgaan
- start de opdracht eenmaal wanneer automatisch ontwaken na voltooiing is ingeschakeld en vertrouw op het pushgebaseerde ontwaakpad
- gebruik `process` voor logboeken, status, invoer of ingrijpen bij een actieve opdracht
- geef voor grotere taken de voorkeur aan `sessions_spawn`; de voltooiing van subagents is pushgebaseerd en wordt automatisch aan de aanvrager gemeld
- poll `subagents list` / `sessions_list` niet herhaaldelijk alleen om op voltooiing te wachten

`agents.defaults.subagents.delegationMode` (standaard `"suggest"`) kan dit versterken. `"prefer"` voegt een speciale sectie **Delegatie aan subagents** toe die de hoofdagent opdraagt als responsieve coördinator op te treden en alles wat uitgebreider is dan een rechtstreeks antwoord via `sessions_spawn` door te sturen. Dit geldt alleen voor de prompt; het toolbeleid bepaalt nog steeds of `sessions_spawn` beschikbaar is.

Veiligheidswaarborgen in de systeemprompt zijn adviserend, niet afdwingend. Gebruik toolbeleid, uitvoeringsgoedkeuringen, sandboxing en kanaaltoelatingslijsten voor harde handhaving; operators kunnen promptwaarborgen bewust uitschakelen.

Bij kanalen met ingebouwde goedkeuringskaarten/-knoppen vertelt de prompt de agent eerst op die UI te vertrouwen en alleen een handmatige opdracht `/approve` op te nemen wanneer het toolresultaat aangeeft dat chatgoedkeuringen niet beschikbaar zijn of handmatige goedkeuring de enige mogelijkheid is.

## Promptmodi

OpenClaw rendert kleinere systeemprompts voor subagents. De runtime stelt per run een `promptMode` in (geen gebruikersgerichte configuratie):

- `full` (standaard): alle bovenstaande secties.
- `minimal`: gebruikt voor subagents; laat de geheugenpromptsectie (meegeleverd als **Geheugen ophalen**), **OpenClaw zelf bijwerken**, **Modelaliassen**, **Gebruikersidentiteit**, **Richtlijnen voor assistentuitvoer**, **Berichten**, **Stille antwoorden** en **Heartbeats** weg. Tooling, **Veiligheid**, **Skills** (indien meegeleverd), Werkruimte, Sandbox, Huidige datum en tijd (indien bekend), Runtime en ingevoegde context blijven beschikbaar.
- `none`: retourneert alleen de basisidentiteitsregel.

Onder `promptMode=minimal` krijgen extra ingevoegde prompts het label **Subagentcontext** in plaats van **Groepschatcontext**.

Voor automatische antwoordruns via kanalen laat OpenClaw de algemene sectie **Stille antwoorden** weg wanneer directe, groeps- of uitsluitend berichttoolcontext al het contract voor zichtbare antwoorden beheert. Alleen de verouderde automatische groeps-/kanaalmodus toont `NO_REPLY`; directe chats en antwoorden uitsluitend via de berichttool slaan richtlijnen voor stille tokens over.

## Promptsnapshots

OpenClaw bewaart vastgelegde promptsnapshots voor het standaardpad van de Codex-runtime onder `test/fixtures/agents/prompt-snapshots/codex-runtime-happy-path/`. Ze renderen geselecteerde thread-/beurtparameters van de appserver plus een gereconstrueerde stapel promptlagen die aan het model is gekoppeld voor directe Telegram-berichten, Discord-groepen en Heartbeat-beurten: een vastgezette fixture voor de Codex-modelprompt `gpt-5.5`, de ontwikkelaarstekst voor machtigingen van het Codex-standaardpad, OpenClaw-ontwikkelaarsinstructies, beurtgebonden instructies voor de samenwerkingsmodus wanneer OpenClaw die levert, gebruikersinvoer voor de beurt en verwijzingen naar dynamische toolspecificaties.

Vernieuw de vastgezette Codex-modelpromptfixture met `pnpm prompt:snapshots:sync-codex-model`. Standaard zoekt deze eerst naar `$CODEX_HOME/models_cache.json`, daarna naar `~/.codex/models_cache.json` en vervolgens naar de onderhoudersconventie voor checkouts `~/code/codex/codex-rs/models-manager/models.json`; als geen daarvan bestaat, wordt afgesloten zonder de vastgelegde fixture te wijzigen. Geef `--catalog <path>` door om vanuit een specifiek bestand `models_cache.json` of `models.json` te vernieuwen.

Deze snapshots zijn geen byte-voor-byteopname van een onbewerkt OpenAI-verzoek. Codex kan runtimebeheerde werkruimtecontext (`AGENTS.md`, omgevingscontext, geheugens, app-/plugininstructies, ingebouwde instructies voor de standaard samenwerkingsmodus) toevoegen nadat OpenClaw thread- en beurtparameters heeft verzonden.

Genereer opnieuw met `pnpm prompt:snapshots:gen`; controleer afwijkingen met `pnpm prompt:snapshots:check`. CI voert de afwijkingscontrole samen met de shards voor aanvullende grenzen uit, zodat promptwijzigingen en snapshotupdates in dezelfde PR worden opgenomen.

## Bootstrapinjectie voor de werkruimte

Bootstrapbestanden worden vanuit de actieve werkruimte bepaald en naar het promptoppervlak geleid dat bij hun levensduur past:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (alleen in gloednieuwe werkruimten)
- `MEMORY.md` indien aanwezig

In de native Codex-harness voorkomt OpenClaw dat stabiele werkruimtebestanden bij elke gebruikersbeurt worden herhaald. Codex laadt `AGENTS.md` via de eigen ontdekking van projectdocumentatie. `TOOLS.md` wordt doorgestuurd als overgenomen Codex-ontwikkelaarsinstructies. `SOUL.md`, `IDENTITY.md` en `USER.md` worden doorgestuurd als beurtgebonden ontwikkelaarsinstructies voor de samenwerkingsmodus, zodat native Codex-subagents deze niet overnemen. De inhoud van `HEARTBEAT.md` wordt niet rechtstreeks ingevoegd; Heartbeat-beurten krijgen een opmerking over de samenwerkingsmodus die naar het bestand verwijst wanneer dit bestaat en niet leeg is. De inhoud van `MEMORY.md` wordt evenmin in elke native Codex-beurt geplakt: wanneer geheugentools voor de werkruimte beschikbaar zijn, krijgen Codex-beurten een korte werkruimtegeheugenopmerking die het model naar `memory_search` of `memory_get` verwijst. Als tools zijn uitgeschakeld, geheugenzoeken niet beschikbaar is of de actieve werkruimte afwijkt van de agentgeheugenwerkruimte, valt `MEMORY.md` terug op het normale begrensde pad voor beurtcontext. `BOOTSTRAP.md` behoudt de normale rol voor beurtcontext.

Op niet-Codex-harnassen worden bootstrapbestanden volgens hun bestaande voorwaarden in de OpenClaw-prompt samengesteld. `HEARTBEAT.md` wordt bij normale runs weggelaten wanneer Heartbeats voor de standaardagent zijn uitgeschakeld of `agents.defaults.heartbeat.includeSystemPromptSection` onwaar is. Houd ingevoegde bestanden beknopt, met name `MEMORY.md` buiten Codex: dit moet een zorgvuldig samengestelde langetermijnsamenvatting blijven, met gedetailleerde dagelijkse notities in `memory/*.md` die naar behoefte kunnen worden opgehaald via `memory_search` / `memory_get`. Te grote `MEMORY.md`-bestanden buiten Codex verhogen het promptgebruik en kunnen gedeeltelijk worden ingevoegd volgens de onderstaande limieten voor bootstrapbestanden.

<Note>
Dagbestanden van `memory/*.md` maken **geen** deel uit van de normale bootstrap-Projectcontext. Tijdens gewone beurten worden ze naar behoefte benaderd via `memory_search` / `memory_get`, zodat ze niet meetellen voor het contextvenster tenzij het model ze expliciet leest. Kale beurten met `/new` en `/reset` vormen de uitzondering: de runtime kan recent dagelijks geheugen als een eenmalig opstartcontextblok vóór die eerste beurt plaatsen.
</Note>

Grote bestanden worden afgekapt met een markering:

| Limiet                                       | Configuratiesleutel                                | Standaard |
| -------------------------------------------- | -------------------------------------------------- | --------- |
| Maximumaantal tekens per bestand             | `agents.defaults.bootstrapMaxChars`                                 | 20000     |
| Totaal voor alle bestanden                   | `agents.defaults.bootstrapTotalMaxChars`                                 | 60000     |
| Waarschuwing bij afkapping (`off`\|`once`\|`always`) | `agents.defaults.bootstrapPromptTruncationWarning` | `always` |

Ontbrekende bestanden voegen een korte markering voor een ontbrekend bestand in. Gedetailleerde aantallen voor onbewerkte/geïnjecteerde inhoud blijven beschikbaar in diagnostiek zoals `/context`, `/status`, doctor en logboeken.

Voor geheugenbestanden betekent afkapping geen gegevensverlies: het bestand blijft intact op schijf. In native Codex wordt `MEMORY.md` indien beschikbaar op aanvraag via geheugentools gelezen, anders wordt een begrensde promptterugval gebruikt. In andere harnesses ziet het model alleen de verkorte geïnjecteerde kopie totdat het het geheugen rechtstreeks leest of doorzoekt. Als `MEMORY.md` herhaaldelijk wordt afgekapt, distilleer het dan tot een kortere duurzame samenvatting, verplaats de gedetailleerde geschiedenis naar `memory/*.md` of verhoog bewust de bootstraplimieten.

Subagentsessies injecteren alleen `AGENTS.md` en `TOOLS.md` (andere bootstrapbestanden worden uitgefilterd om de subagentcontext klein te houden).

Interne hooks kunnen deze stap onderscheppen via de gebeurtenis `agent:bootstrap` om de geïnjecteerde bootstrapbestanden te wijzigen of te vervangen (bijvoorbeeld door `SOUL.md` om te wisselen voor een alternatieve persona).

Begin met de [persoonlijkheidsgids in SOUL.md](/nl/concepts/soul) om minder algemeen te klinken.

Gebruik `/context list` of `/context detail` om te bekijken hoeveel elk geïnjecteerd bestand bijdraagt (onbewerkt versus geïnjecteerd, afkapping, overhead van toolschema's). Zie [Context](/nl/concepts/context).

## Tijdverwerking

De sectie **Huidige datum en tijd** verschijnt alleen wanneer de tijdzone van de gebruiker bekend is en bevat alleen de **tijdzone** (geen dynamische klok of tijdnotatie), zodat de promptcache stabiel blijft.

Gebruik `session_status` wanneer de agent de huidige tijd nodig heeft; de statuskaart ervan bevat een regel met een tijdstempel. Dezelfde tool kan optioneel een modeloverschrijving per sessie instellen (`model=default` wist deze).

Configureer met:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Zie [Tijdzones](/nl/concepts/timezone) en [Datum en tijd](/nl/date-time) voor volledige details over het gedrag.

## Skills

Wanneer er geschikte Skills bestaan, injecteert OpenClaw een compacte lijst `<available_skills>` (`formatSkillsForPrompt`) met het **bestandspad** en per Skill een van de inhoud afgeleide markering `<version>sha256:...</version>`. De prompt instrueert het model om `read` te gebruiken om het bestand SKILL.md op de vermelde locatie (werkruimte, beheerd of meegeleverd) te laden en om een Skill opnieuw te lezen wanneer de waarde van `<version>` afwijkt van die in een eerdere beurt. Als er geen Skills geschikt zijn, wordt de sectie Skills weggelaten.

Native Codex-beurten ontvangen deze lijst als samenwerkingsinstructies voor ontwikkelaars die alleen voor die beurt gelden, in plaats van als gebruikersinvoer per beurt, met uitzondering van lichtgewicht Cron-beurten die de exacte geplande prompt behouden. Andere harnesses behouden de normale promptsectie.

De locatie kan naar een geneste Skill verwijzen, zoals `skills/personal/foo/SKILL.md`. De nesting dient alleen voor organisatie; de prompt gebruikt de platte Skill-naam uit de frontmatter van `SKILL.md`.

Geschiktheid omvat poorten voor Skill-metadata, controles van de runtimeomgeving/configuratie en de effectieve allowlist voor agentskills wanneer `agents.defaults.skills` of `agents.entries.*.skills` is geconfigureerd. Met Plugins meegeleverde Skills zijn alleen geschikt wanneer hun eigenaar-Plugin is ingeschakeld, zodat tool-Plugins uitgebreidere bedieningshandleidingen kunnen aanbieden zonder al die richtlijnen in elke toolbeschrijving op te nemen.

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
    <version>sha256:...</version>
  </skill>
</available_skills>
```

Hierdoor blijft de basisprompt klein, terwijl gericht gebruik van Skills mogelijk blijft. De dimensionering wordt beheerd door het Skills-subsysteem, los van de algemene dimensionering voor lezen/injecteren tijdens runtime:

| Bereik     | Promptbudget voor Skills                             | Budget voor runtimefragmenten      |
| --------- | ---------------------------------------------------- | ---------------------------------- |
| Globaal   | `skills.limits.maxSkillsPromptChars`                 | `agents.defaults.contextLimits.*`  |
| Per agent | `agents.entries.*.skillsLimits.maxSkillsPromptChars` | `agents.entries.*.contextLimits.*` |

Het budget voor runtimefragmenten omvat `memory_get`, live toolresultaten en vernieuwingen van `AGENTS.md` na Compaction.

## Documentatie

De sectie **Documentatie** verwijst naar lokale documentatie wanneer die beschikbaar is (`docs/` in een Git-checkout of de documentatie van het meegeleverde npm-pakket), en valt anders terug op [https://docs.openclaw.ai](https://docs.openclaw.ai). Deze sectie vermeldt ook de locatie van de OpenClaw-broncode: Git-checkouts tonen de lokale hoofdmap van de broncode; pakketinstallaties krijgen de GitHub-URL van de broncode, met instructies om de broncode daar te raadplegen wanneer de documentatie onvolledig of verouderd is.

De prompt presenteert de documentatie als de gezaghebbende bron voor zelfkennis over OpenClaw voordat het model begrijpt hoe OpenClaw werkt (geheugen/dagelijkse notities, sessies, tools, Gateway, configuratie, opdrachten, projectcontext), en instrueert het model om `AGENTS.md`, projectcontext, werkruimte-/profiel-/geheugennotities en `memory_search` te behandelen als instructiecontext of gebruikersgeheugen, en niet als kennis over het ontwerp/de implementatie van OpenClaw. Als de documentatie niets vermeldt of verouderd is, moet het model dit aangeven en de broncode inspecteren. De prompt instrueert het model ook om `openclaw status` indien mogelijk zelf uit te voeren en de gebruiker alleen te vragen wanneer het geen toegang heeft.

Specifiek voor configuratie verwijst de prompt agents naar de toolactie `config.schema.lookup` van `gateway` voor exacte documentatie en beperkingen op veldniveau, en vervolgens naar `docs/gateway/configuration.md` en `docs/gateway/configuration-reference.md` voor bredere richtlijnen.

## Gerelateerd

- [Agentruntime](/nl/concepts/agent)
- [Agentwerkruimte](/nl/concepts/agent-workspace)
- [Context-engine](/nl/concepts/context-engine)
