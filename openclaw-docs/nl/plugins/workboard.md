---
read_when:
    - Je wilt een werkbord in Kanban-stijl in de Control UI
    - Je schakelt de gebundelde Workboard-plugin in of uit
    - Je wilt gepland agentwerk bijhouden zonder een externe projectmanager
summary: Optioneel dashboardwerkbord voor kaarten in beheer van agents en sessieoverdracht
title: Workboard-Plugin
x-i18n:
    generated_at: "2026-07-27T06:04:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec05c990c3559015780d9cb80f3ceedd7cc79db89ccf1afd65906c8c7630331
    source_path: plugins/workboard.md
    workflow: 16
---

De Workboard-plugin voegt een optioneel bord in Kanban-stijl toe aan de
[Control UI](/nl/web/control-ui): werkkaarten op agentformaat, toewijzing aan agents
en een koppeling terug naar de taak, run en dashboardsessie van de kaart.

Workboard is bewust klein: het houdt lokaal operationeel werk bij voor één
OpenClaw Gateway. Het is geen vervanging voor GitHub Issues, Linear, Jira of
andere projectbeheersystemen voor teams.

## Inschakelen

Workboard is gebundeld, maar standaard uitgeschakeld:

1. Open **Plugins** in de Control UI of gebruik `/settings/plugins` ten opzichte van
   het geconfigureerde basispad van de Control UI. Een basispad van `/openclaw`
   gebruikt bijvoorbeeld `/openclaw/settings/plugins`.
2. Zoek **Workboard** en kies **Enable**. Omdat Workboard bij
   OpenClaw is inbegrepen, is geen **Install**-actie nodig.
3. Als de UI meldt dat opnieuw opstarten vereist is, start je de Gateway opnieuw.

Het tabblad Workboard verschijnt in de dashboardnavigatie nadat de pluginruntime is geladen.
Zolang de plugin is uitgeschakeld, blijft het tabblad verborgen in de navigatie. Als je de
route `/workboard` rechtstreeks opent terwijl de plugin is uitgeschakeld of wordt geblokkeerd door
`plugins.allow`/`plugins.deny`, wordt een status weergegeven waarin de plugin niet beschikbaar is, in plaats van
kaartgegevens.

De equivalente CLI-workflow is:

```bash
openclaw plugins enable workboard
openclaw gateway restart
openclaw dashboard
```

## Configuratie

Workboard heeft geen pluginspecifieke configuratie. Schakel de plugin in of uit met de standaard
pluginvermelding:

```json5
{
  plugins: {
    entries: {
      workboard: {
        enabled: true,
        config: {},
      },
    },
  },
}
```

```bash
openclaw plugins disable workboard
openclaw gateway restart
```

## Kaartvelden

| Veld        | Waarden                                                                                                       |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `status`    | `triage`, `backlog`, `todo`, `scheduled`, `ready`, `running`, `review`, `blocked`, `done`                     |
| `priority`  | `low`, `normal`, `high`, `urgent`                                                                             |
| `labels`    | vrije tekst                                                                                                   |
| `agentId`   | optioneel toegewezen agent                                                                                    |
| gekoppelde verwijzingen | optionele taak, run, sessie of bron-URL                                                                    |
| `execution` | optionele metagegevens voor een Codex-/Claude-run die vanaf de kaart is gestart (engine, modus, model, sessie, run-id, status) |

Kaarten bevatten ook compacte metagegevens voor pogingen, opmerkingen, koppelingen, bewijs,
artefacten, automatiseringsinstellingen, bijlagen, workerlogboeken, workerprotocolstatus,
claims, diagnostiek, meldingen, sjabloon-id, archiefstatus en
detectie van verouderde sessies, plus een lijst met recente gebeurtenissen (`created`, `edited`,
`moved`, `linked`, `specified`, `decomposed`, `claimed`, `heartbeat`,
`execution_updated`, `attempt_started`, `attempt_updated`, `comment_added`,
`link_added`, `proof_added`, `artifact_added`, `attachment_added`,
`diagnostic`, `notification`, `dispatch`, `orchestration`,
`protocol_violation`, `archived`, `unarchived`, `stale`). Met deze metagegevens kan een
operator zien hoe een kaart door het bord is verplaatst zonder de gekoppelde
sessie te openen; het is lokale operationele context, geen vervanging voor
sessietranscripten of de geschiedenis van GitHub-issues.

De plugin en de Control UI gebruiken één Workboard-kaartcontract. Dashboardvernieuwingen
behouden daarom de herkomst en autoriteit van de werkruimte, de claimstatus, diagnostische
acties en volgnummers van meldingen, in plaats van een kleinere
kopie van de kaart te projecteren die alleen voor de UI bestemd is. Onbekende diagnostische typen, diagnostische ernstniveaus en
meldingstypen worden genegeerd totdat beide oppervlakken ze ondersteunen; ze worden nooit
herschreven naar een andere geldige status.

Het geopende dashboard wordt bijgewerkt via `plugin.workboard.changed`-invalidaties. Elke
gebeurtenis bevat alleen een store-epoch en revisie; de UI leest vervolgens de canonieke
kaarten opnieuw via de normale `operator.read`-RPC. Meerdere revisies worden samengevoegd tot
één vervolgleesactie. Workboard stelt die leesactie uit terwijl een kaart wordt versleept,
bewerkt of geschreven en hervat deze nadat de lokale interactie is voltooid. Bij
opnieuw verbinden wordt altijd een canonieke herlaadactie uitgevoerd. Er vindt geen routinematige volledige-kaartpolling
plaats en **Refresh** blijft beschikbaar voor handmatig herstel.

Wanneer er meer dan één bord bestaat, bevat de werkbalk een **Board**-filter dat wordt ondersteund
door permanente bordmetagegevens en niet alleen door de momenteel zichtbare kaarten. Lege
en gearchiveerde borden blijven daardoor selecteerbaar. Kaarten zonder expliciete
bord-id behoren tot het canonieke bord `default`. Elk bord heeft een canonieke
pagina `/workboard/<boardId>` die als bladwijzer kan worden opgeslagen, gedeeld of vastgezet in de
zijbalk. De eerder uitgebrachte vorm `/workboard?board=<boardId>` blijft een
compatibiliteitsalias en leidt door naar die pagina, waarbij andere queryparameters
behouden blijven. Als je **All boards** kiest, keer je terug naar `/workboard`.

Kaarten worden opgeslagen in de eigen Gateway-status van de plugin en worden samen met de rest van
de OpenClaw-status van die Gateway verplaatst (zie [Opslag](#storage)).

## Werk starten vanaf een kaart

Niet-gekoppelde kaarten kunnen rechtstreeks werk starten:

- **Run Codex** / **Run Claude** start een door een taak gevolgde agent-run met een
  expliciete engine, verzendt de kaartprompt en markeert de kaart als `running`. Codex-
  runs gebruiken `openai/gpt-5.6-sol`; Claude-runs gebruiken `anthropic/claude-sonnet-4-6`.
- **Open Codex** / **Open Claude** maakt een gekoppelde dashboardsessie zonder
  de kaartprompt te verzenden of de kaart te verplaatsen, voor handmatig werk dat aan
  het bord gekoppeld blijft.

Autonome starts gebruiken het pad voor door taken gevolgde agent-runs van de Gateway (standaardagent
en -model, tenzij Codex/Claude expliciet wordt gekozen); Workboard koppelt vervolgens de
resulterende taak, run-id en sessiesleutel terug aan de kaart. Elke gekoppelde
uitvoering registreert ook een samenvatting van de poging (engine, modus, model, run-id,
tijdstempels, status, doorlopend aantal mislukkingen), zodat herhaalde mislukkingen zichtbaar blijven.

Het dashboard vernieuwt de taakstatus vanuit het taaktboek van de Gateway en koppelt
taken aan kaarten op basis van taak-id, run-id of gekoppelde sessiesleutel. Een taak die in de wachtrij staat of wordt uitgevoerd,
houdt de levenscyclus van de kaart actief; een voltooide, mislukte, verlopen of
geannuleerde taak verplaatst de kaart richting `review` of `blocked` volgens dezelfde synchronisatieregel
als gekoppelde sessies (zie [Synchronisatie van de sessielevenscyclus](#session-lifecycle-sync)).

## Agent-tools

| Tool                                                                                                                                             | Doel                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workboard_list`                                                                                                                                 | Compacte kaarten met claim-/diagnostische status weergeven; optioneel filter op bord.                                                                                                     |
| `workboard_read`                                                                                                                                 | Eén kaart plus begrensde workercontext retourneren (notities, pogingen, opmerkingen, links, bewijs, artefacten, bovenliggende resultaten, recent werk van de toegewezene, actieve diagnostiek). |
| `workboard_create`                                                                                                                               | Een kaart maken met optionele bovenliggende kaarten, tenant, Skills, bord, werkruimtemetadata, idempotentiesleutel, runtimelimiet en budget voor nieuwe pogingen.                           |
| `workboard_link`                                                                                                                                 | Een bovenliggende kaart aan een onderliggende kaart koppelen. Onderliggende kaarten blijven `todo` totdat elke bovenliggende kaart `done` bereikt; daarna verplaatst dispatchpromotie ze naar `ready`. |
| `workboard_claim`                                                                                                                                | Een kaart claimen voor de aanroepende agent; verplaatst `backlog`/`todo`/`ready` naar `running`.                                                  |
| `workboard_heartbeat`                                                                                                                            | De Heartbeat van de claim tijdens een langere uitvoering vernieuwen.                                                                                                                      |
| `workboard_release`                                                                                                                              | De claim na voltooiing, pauzering of overdracht vrijgeven; kan de kaart naar een volgende status verplaatsen.                                                                              |
| `workboard_complete` / `workboard_block`                                                                                                         | Gestructureerde levenscyclustools voor eindsamenvattingen, bewijs, artefacten en manifesten van gemaakte kaarten (moeten verwijzen naar kaarten die aan de voltooide kaart zijn teruggekoppeld) of redenen voor blokkering. |
| `workboard_attachment_add` / `workboard_attachment_read` / `workboard_attachment_delete`                                                         | Kleine kaartbijlagen opslaan in de SQLite-status van de Plugin, op de kaart indexeren en in de workercontext beschikbaar stellen.                                                          |
| `workboard_worker_log` / `workboard_protocol_violation`                                                                                          | Workerlogregels vastleggen en een kaart blokkeren wanneer een geautomatiseerde worker stopt zonder `workboard_complete`/`workboard_block` aan te roepen.                                  |
| `workboard_board_create` / `workboard_board_archive` / `workboard_board_delete`                                                                  | Persistente bordmetadata beheren (weergavenaam, beschrijving, archiefstatus, standaardwerkruimte).                                                                                        |
| `workboard_runs`                                                                                                                                 | De persistente geschiedenis van uitvoeringspogingen voor een kaart retourneren.                                                                                                          |
| `workboard_specify`                                                                                                                              | Een ruwe triage-/backlogkaart omzetten in een verduidelijkte `todo`-kaart; legt de specificatiesamenvatting op de kaart vast.                                                  |
| `workboard_decompose`                                                                                                                            | Een bovenliggende orchestratiekaart opsplitsen in gekoppelde onderliggende kaarten die bord-/tenantmetadata overnemen; kan de bovenliggende kaart voltooien met een manifest van gemaakte kaarten. |
| `workboard_notify_subscribe` / `workboard_notify_list` / `workboard_notify_events` / `workboard_notify_advance` / `workboard_notify_unsubscribe` | Abonnementen op meldingen beheren. Gebeurtenissen kunnen veilig opnieuw worden gelezen; `advance` verplaatst de duurzame cursor, zodat aanroepers kunnen hervatten zonder gebeurtenissen van voltooide/mislukte/verouderde kaarten te verliezen of dubbel te lezen. |
| `workboard_boards` / `workboard_stats`                                                                                                           | Bordnaamruimten en wachtrijstatistieken inspecteren.                                                                                                                                      |
| `workboard_promote` / `workboard_reassign` / `workboard_reclaim`                                                                                 | Vastgelopen werk herstellen of overdragen.                                                                                                                                                |
| `workboard_comment` / `workboard_proof`                                                                                                          | Overdrachtsnotities toevoegen of verwijzingen naar bewijs/artefacten bijvoegen.                                                                                                           |
| `workboard_unblock`                                                                                                                              | Geblokkeerd werk terugverplaatsen naar `todo`.                                                                                                                                |
| `workboard_move`                                                                                                                                 | Een kaart naar een andere status verplaatsen; voor geclaimde kaarten is de agentclaimscope van de aanroeper vereist.                                                                      |
| `workboard_dispatch`                                                                                                                             | Afhankelijkheidspromotie of opschoning van verouderde claims activeren zonder workers te starten; workers worden gestart via Gateway- of slash-commandodispatch.                          |

Bewijsstatussen zijn door workers gerapporteerde uitkomsten, geen onafhankelijke verificatie. Een `passed`-vermelding betekent dat de worker meldt dat de opdracht of controle is geslaagd; gebruikers die een onafhankelijke kwaliteitscontrole nodig hebben, moeten de bijgevoegde opdracht, URL of het artefact inspecteren en hun eigen verificatie uitvoeren. `workboard_proof` retourneert de `proofId` van de nieuwe record. Wanneer `workboard_complete` de terminale status van datzelfde bewijs rapporteert, geef je `proofId` door, zodat de openstaande record ter plaatse wordt afgehandeld zonder zijn identiteit of tijdstempel te verliezen. Bewijs dat al dezelfde terminale status heeft, wordt ongewijzigd hergebruikt. Voltooiingsbewijs zonder `proofId` blijft alleen toevoegbaar, zodat een latere nieuwe poging oudere geschiedenis niet kan herschrijven enkel omdat de opdracht of notitie identiek is.

Geclaimde kaarten weigeren mutaties door agenttools van andere agents, tenzij de aanroeper het claimtoken bezit dat door `workboard_claim` is geretourneerd. Elke kaart die door een agenttool of Gateway-RPC-aanroep wordt geretourneerd, redigeert `metadata.claim.token` tot `[redacted]` (het token zelf wordt eenmalig, op het hoogste niveau en uitsluitend door `workboard_claim` geretourneerd), zodat dashboardbeheerders en andere agents de claimstatus kunnen inspecteren zonder ooit een bruikbaar token te zien. Herstel verloopt via `workboard_promote`/`workboard_reassign`/`workboard_reclaim`, waarvoor het token niet vereist is.

## Dispatch

Dispatch is lokaal voor de Gateway: er worden geen willekeurige OS-processen gestart. Normale OpenClaw-subagentsessies blijven verantwoordelijk voor de uitvoering. Eén dispatchdoorgang:

1. Promoveert kaarten waarvan de afhankelijkheden gereed zijn.
2. Legt dispatchmetadata vast op gereedstaande kaarten.
3. Blokkeert verlopen claims of uitvoeringen waarvan de time-out is verstreken.
4. Markeert door het bord geconfigureerde triagekaarten als orchestratiekandidaten.
5. Claimt een kleine batch gereedstaande kaarten en start workeruitvoeringen via de subagentruntime van de
   Gateway.

Workers krijgen een begrensde kaartcontext plus het claimtoken dat nodig is om via de Workboard-tools een Heartbeat te verzenden of de kaart te voltooien of blokkeren.

Werkruimtepaden volgen de bestaande bestandssysteembevoegdheden van de aanroeper. Gateway-clients met `operator.write` kunnen geconfigureerde agentwerkruimten gebruiken; `operator.admin`-clients kunnen andere host-check-outs gebruiken. Agenttools in een sandbox gebruiken de werkruimtetoegang van hun sandbox, terwijl niet-gesandboxte tools die uitsluitend voor werkruimten bestemd zijn hun geconfigureerde werkruimteroot gebruiken. Workboard legt die bevoegdheid vast wanneer een werkruimte wordt toegewezen en doorsnijdt deze tijdens dispatch opnieuw met de bevoegdheid van de huidige aanroeper, zodat een persistente kaart de toegang van een latere aanroeper niet kan uitbreiden. Bij oudere kaarten met een expliciete hostwerkruimte maar zonder vastgelegde bevoegdheid moet die werkruimte opnieuw worden opgeslagen voordat dispatch met volledige hosttoegang mogelijk is; kaarten zonder hostpad nemen bij hun eerste dispatch de bevoegdheid van de huidige aanroeper over.

Werkruimtegebonden dispatch accepteert een map of Git-check-out alleen wanneer de root van de repository exact overeenkomt met de doelwerkruimte van de agent. Een worktree-aanvraag wordt beperkt tot die map en als mapwerkruimte opgeslagen, zodat de host de check-out niet materialiseert of instelcode van de repository uitvoert. De doelworker moet voor precies die werkruimte een schrijfbare, niet-gedeelde Docker-sandbox gebruiken, zonder uitvoering met verhoogde bevoegdheden, persistente overrides voor uitvoering op de host/Node of niet-geclassificeerde Plugin- en MCP-tools. Workboard somt zijn geregistreerde tools op in plaats van een `workboard_*`-voorvoegsel te vertrouwen, en dispatch weigert een actieve Docker-container waarvan de live hash van mount/configuratie verouderd is. Dispatch rapporteert het incompatibele doelbeleid in plaats van een minder strikt geïsoleerde worker te starten. Dispatch met volledige hosttoegang kan andere lokale check-outs als doel gebruiken en behoudt de normale beheerde worktree-instelling.

Werkruimtebevoegdheid creëert geen tweede machtigingsmodel voor de levenscyclus van kaarten. Aanroepers die Workboard-kaarten mogen muteren, kunnen deze op elk oppervlak handmatig door dezelfde statussen verplaatsen; alleen-lezen toegang tot een werkruimte voorkomt uitsluitend workerdispatch waarvoor schrijfrechten nodig zijn.

### Workerselectie

Elke cyclus start **standaard maximaal 3 workers**. Gereedstaande kaarten worden gesorteerd op
prioriteit, vervolgens positie en daarna aanmaaktijd. Een cyclus start slechts één kaart per
eigenaar/agent en slaat eigenaren over die al actief werk of reviewwerk op het
bord hebben. Gearchiveerde kaarten, kaarten met een actieve claim en kaarten die niet de status `ready`
hebben, worden nooit geselecteerd om workers te starten (ze kunnen nog steeds worden beïnvloed door de
gegevenszijde van dispatch: verouderde claims opschonen, afhankelijkheden promoveren en time-outs
opschonen).

Sessiesleutels zijn deterministisch per bord/kaart, zodat herhaalde dispatches
teruggaan naar dezelfde workerlane in plaats van niet-gerelateerde sessies te maken:

- Toegewezen kaarten: `agent:<agentId>:subagent:workboard-<boardId>-<cardId>`
- Niet-toegewezen kaarten: `subagent:workboard-<boardId>-<cardId>` (Gateway bepaalt
  de geconfigureerde standaardagent)

Als een worker niet kan worden gestart nadat een kaart is geclaimd, blokkeert Workboard de
kaart, verwijdert het de claim, registreert het de fout bij het starten van de run en voegt het een
workerlogregel toe, zichtbaar in het dashboard, de CLI-JSON, agenttools en de
kaartdiagnostiek.

### Toegangspunten

- Dispatchactie in het dashboard
- `openclaw workboard dispatch`
- `/workboard dispatch` in een kanaal dat opdrachten ondersteunt

Alle drie gebruiken de subagentruntime van de Gateway wanneer de Gateway beschikbaar is. De
CLI heeft één fallback voor operators: als de Gateway-aanroep mislukt met een
verbindings-/onbeschikbaarheidsfout (of een `unknown method`-fout voor oudere
Gateways), er geen expliciet `--url`-/`--token`-doel is en er geen geconfigureerde externe
Gateway (`OPENCLAW_GATEWAY_URL` of `gateway.mode: remote`) van toepassing is, voert de CLI
een dispatch uit die alleen gegevens verwerkt op basis van de lokale SQLite-status. Deze kan afhankelijkheden promoveren,
verouderde claims opschonen en runs met een time-out blokkeren, maar kan geen workers starten. Authenticatie-,
machtigings- en validatiefouten van een bereikbare Gateway worden niet als
onbeschikbaarheid behandeld; ze worden weergegeven als opdrachtfouten. Hetzelfde geldt voor elke Gateway-
fout wanneer een expliciet `--url`-/`--token`-doel is opgegeven.

Bordmetadata kan `autoDecompose`, `autoDecomposePerDispatch`,
`defaultAssignee` en `orchestratorProfile` instellen. OpenClaw registreert deze intentie en
maakt die beschikbaar in de workercontext; de daadwerkelijke specificatie/opsplitsing verloopt nog steeds
via de normale Workboard-tools.

## CLI en slashopdracht

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create "Verouderde levenscyclus van kaart herstellen" --priority high --labels bug,workboard
openclaw workboard show <card-id> [--json]
openclaw workboard move <card-id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--json]
```

`list`-tekstuitvoer verbergt standaard gearchiveerde kaarten (`--include-archived`
overschrijft dit); `--json` bevat altijd gearchiveerde kaarten, overeenkomstig het contract voor volledige kaarten
dat bestaande scripts gebruiken. `show` en `move` accepteren een ondubbelzinnig id-
voorvoegsel. `list`, `create`, `show` en `move` lezen/schrijven altijd rechtstreeks
de lokale Pluginstatus. Alleen `dispatch` roept de actieve Gateway aan, met de hierboven
beschreven fallback.

Zie [Workboard-CLI](/nl/cli/workboard) voor alle vlaggen, JSON-uitvoer, Gateway-
fallbackgedrag, verwerking van id-voorvoegsels, selectieregels voor dispatch en
probleemoplossing.

`/workboard list`, `/workboard show <card-id>`, `/workboard create <title>`,
`/workboard move <card-id> --status <status>` en `/workboard dispatch` weerspiegelen
de CLI. Weergeven en tonen zijn leesbewerkingen voor elke gemachtigde afzender van opdrachten.
Maken, verplaatsen en dispatch vereisen de eigenaarsstatus op chatinterfaces, of een Gateway-
client met `operator.write`/`operator.admin`. Handmatige verplaatsingen door operators gebruiken hetzelfde
gedrag voor het overschrijven van claims als slepen en neerzetten in het dashboard. De toegang tot de worktree
volgt nog steeds dezelfde hierboven beschreven werkruimtegrens.

## Synchronisatie van de sessielevenscyclus

Kaarten kunnen worden gekoppeld aan een bestaande dashboardsessie of aan een sessie die wordt gemaakt wanneer je
werk vanuit de kaart start. Gekoppelde kaarten tonen de sessielevenscyclus inline:
actief, verouderd, gekoppeld en inactief, voltooid, mislukt of ontbrekend. Je kunt ook een
bestaande sessie vastleggen vanaf het tabblad Sessions met **Add to Workboard**; de kaart
wordt aan die sessie gekoppeld, gebruikt het sessielabel of de recente gebruikersprompt als titel
en vult notities vooraf in met de recente gebruikersprompt plus het laatste antwoord van de assistent,
indien beschikbaar.

Als de gekoppelde sessie ontbreekt, blijft de kaart voor context gekoppeld en
biedt deze nog steeds startbesturingselementen om opnieuw te starten in een nieuwe sessie. Als een actieve
gekoppelde sessie geen recente activiteit meer rapporteert, markeert Workboard de kaart als
`stale` en slaat dit op als metadata totdat de levenscyclus het wist.

Zolang een kaart zich in een actieve werkstatus bevindt, volgt Workboard de gekoppelde sessie:

| Status van gekoppelde sessie          | Kaartstatus |
| ------------------------------------- | ----------- |
| actief                                | `running`   |
| voltooid                              | `review`    |
| mislukt, beëindigd, time-out of afgebroken | `blocked`   |

**Handmatige reviewstatussen hebben voorrang.** Als je een kaart verplaatst naar `review`, `blocked` of `done`,
stopt de automatische synchronisatie voor die kaart totdat je deze terugverplaatst naar `todo` of `running`.

Het starten van een kaart gebruikt normale Gateway-sessies; Workboard slaat alleen kaart-
metadata en koppelingen op. Het gesprekstranscript, de modelselectie en de run-
levenscyclus blijven in beheer van het reguliere sessiesysteem. Gebruik **Stop** op een actieve
gekoppelde kaart om de actieve run af te breken. Workboard markeert die kaart als `blocked`, zodat
deze zichtbaar blijft voor opvolging.

Nieuwe kaarten kunnen beginnen met Workboard-sjablonen (`bugfix`, `docs`, `release`,
`pr_review`, `plugin`). Sjablonen vullen titel, notities, labels en prioriteit vooraf in;
de sjabloon-id wordt opgeslagen als kaartmetadata.

## Dashboardworkflow

1. Open het tabblad Workboard in de Control UI.
2. Maak een kaart met een titel, notities, prioriteit, labels, een optionele agent en
   een optioneel gekoppelde sessie, of open Sessions en kies **Add to Workboard**
   voor een bestaande sessie.
3. Sleep de kaart tussen kolommen, of focus het compacte statusbesturingselement en gebruik
   het menu of ArrowLeft/ArrowRight. Tijdens het slepen wordt de bronkaart gedimd en
   krijgen beschikbare doelkolommen een omlijning.
4. Start werk vanuit de kaart om een dashboardsessie te maken of opnieuw te gebruiken.
5. Open de gekoppelde sessie vanuit de kaart terwijl de agent werkt.
6. Laat de levenscyclussynchronisatie actief werk verplaatsen naar `review`/`blocked` en verplaats
   de kaart vervolgens handmatig naar `done` wanneer deze is geaccepteerd.

### Sessiebordwidgets

Workboard levert twee native widgets voor sessiedashboards (zie
[Dashboards](/web/dashboards)). De agent maakt ze vast met de tool `dashboard`
met `content: { kind: "plugin", pluginKind, props }`, waarna ze worden weergegeven als
eigen UI met livegegevens, zonder sandboxframe of toekenning van mogelijkheden:

- `workboard:card` met `props: { cardId }` toont één kaart met het status-
  besturingselement, de prioriteit en de toegewezen agent.
- `workboard:mini` met optioneel `props: { boardId, limit }` toont aantallen
  per status plus de belangrijkste gereedstaande/actieve kaarten en koppelt naar de volledige bordpagina.
  Zonder `boardId` worden alle borden samengevoegd; met `boardId` wordt het bereik beperkt tot dat
  bord (kaarten die zonder expliciete bord-id zijn gemaakt, staan op `default`).

## Diagnostiek

Diagnostiek wordt berekend op basis van lokale kaartmetadata. Ingebouwde controles signaleren:

| Soort                       | Voorwaarde                                                                      |
| --------------------------- | ------------------------------------------------------------------------------ |
| `stranded_ready`            | Toegewezen `todo`-/`backlog`-/`ready`-kaart is al meer dan 1 uur niet bijgewerkt.             |
| `running_without_heartbeat` | `running`-kaart zonder claim-Heartbeat of uitvoeringsupdate gedurende meer dan 20 minuten. |
| `blocked_too_long`          | `blocked`-kaart is al meer dan 24 uur niet bijgewerkt.                                   |
| `repeated_failures`         | Het bijgehouden aantal fouten van de kaart is 2 of meer.                                |
| `missing_proof`             | `done`-kaart zonder bewijs, artefacten of bijlagen.                          |
| `orphaned_session`          | `running`-kaart met een `sessionKey`, maar zonder `execution`-metadata.                |

## Machtigingen

Gateway-RPC-methoden vallen onder `workboard.*`:

| Bereik           | Methoden                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`  | `cards.list`, `cards.export`, `cards.diagnostics`, bijlagen weergeven/ophalen, notificatiegebeurtenissen lezen, `boards.list`, `cards.stats`, `cards.runs`                                                                                                                                                                                                                                       |
| `operator.write` | `cards.diagnostics.refresh`, maken/bijwerken/verplaatsen/verwijderen/reageren/koppelen/afhankelijkheid koppelen/bewijs/artefact, bijlage toevoegen/verwijderen, workerlog, protocolschending, claimen/Heartbeat/vrijgeven/promoveren/opnieuw toewijzen/terugvorderen/voltooien/blokkeren/deblokkeren, `cards.dispatch`, `cards.bulk`, archiveren, `boards.upsert`/`archive`/`delete`, `cards.specify`/`decompose`, notificatie abonneren/verwijderen/vooruitzetten |

Geen enkele RPC-methode vereist `operator.admin`. Browsers die zijn verbonden met alleen-lezen
operatortoegang kunnen het bord bekijken, maar kunnen kaarten niet wijzigen. Een beheerdersbereik
verruimt de geaccepteerde Workboard-hostpaden; het wijzigt de beschikbare methoden niet.

## Opslag

Workboard slaat duurzame gegevens op in een relationele SQLite-database die eigendom is van de Plugin
onder de OpenClaw-statusmap: borden, kaarten, labels, levenscyclusgebeurtenissen,
runpogingen, opmerkingen, afhankelijkheidskoppelingen, bewijs, artefactverwijzingen,
bijlagemetadata en -blobs, diagnostiek, notificaties, workerlogs,
protocolstatus en abonnementen bevinden zich allemaal in Workboard-tabellen (niet in
sleutel-waarde-items van de Plugin). Een kaart-export behoudt het bordverhaal
zonder de inhoud van bijlageblobs inline op te nemen.

Installaties die Workboard in de release `.28` gebruikten, kunnen
`openclaw doctor --fix` uitvoeren om de meegeleverde verouderde naamruimten voor Pluginstatus
(`workboard.cards`, `workboard.boards`, `workboard.notify` en, indien aanwezig,
`workboard.attachments`) naar de relationele database te migreren.

## Probleemoplossing

**Het tabblad meldt dat Workboard niet beschikbaar is**

```bash
openclaw plugins inspect workboard --runtime --json
```

Als `plugins.allow` is geconfigureerd, voeg je `workboard` eraan toe. Als `plugins.deny`
`workboard` bevat, verwijder je dit voordat je de Plugin inschakelt.

**Kaarten worden niet opgeslagen**

Controleer of de browserverbinding `operator.write`-toegang heeft. Alleen-lezen operator-
sessies kunnen kaarten weergeven, maar kunnen ze niet maken, bewerken, verplaatsen of verwijderen.

**Het starten van een kaart opent niet de verwachte sessie**

Controleer de agent-id en gekoppelde sessie van de kaart en open vervolgens Sessions of Chat om
de werkelijke runstatus te bekijken.

**Dispatch start geen worker**

Controleer of er ten minste één `ready`-kaart zonder actieve claim is:

```bash
openclaw workboard list --status ready
```

Als de CLI alleen-gegevensdispatch meldt, start of herstart je de Gateway en
probeer je het opnieuw. Alleen-gegevensdispatch werkt de lokale bordstatus bij, maar kan geen
subagentworkerruns starten. Kaarten kunnen ook worden overgeslagen wanneer een andere kaart voor
dezelfde eigenaar of agent al wordt uitgevoerd of op review wacht; voltooi,
blokkeer of geef dat actieve werk vrij voordat je meer werk voor dezelfde
eigenaar dispatcht.

## Gerelateerd

- [Control-UI](/nl/web/control-ui)
- [Werkbord-CLI](/nl/cli/workboard)
- [Plugins](/nl/tools/plugin)
- [Plugins beheren](/nl/plugins/manage-plugins)
- [Sessies](/nl/concepts/session)
