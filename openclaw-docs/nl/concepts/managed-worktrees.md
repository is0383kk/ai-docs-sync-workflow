---
read_when:
    - Je wilt een geïsoleerde branch en checkout voor een agenttaak
    - Je configureert Workboard-kaarten met worktree-werkruimten
    - Je moet een door OpenClaw beheerde worktree herstellen of opschonen
summary: Voer agenttaken uit in geïsoleerde git-check-outs met automatische snapshots en opschoning
title: Beheerde worktrees
x-i18n:
    generated_at: "2026-07-27T05:42:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98ed2579b7243544dbdb550c4b8a292ccd4ab494fd4a45b2404256691c831401
    source_path: concepts/managed-worktrees.md
    workflow: 16
---

Beheerde worktrees geven een agenttaak een eigen git-branch en checkout zonder tijdelijke mappen in de bronrepository te plaatsen. OpenClaw maakt ze aan onder de eigen statusmap, registreert ze in de gedeelde statusdatabase en maakt vóór verwijdering snapshots van de bijgehouden en niet-genegeerde, niet-bijgehouden inhoud.

## Indeling en namen

Elke worktree bevindt zich op:

```text
<openclaw-state-dir>/worktrees/<repo-fingerprint>/<name>
```

De repositoryvingerafdruk bestaat uit de eerste 16 hexadecimale tekens van een SHA-256-hash van de canonieke gemeenschappelijke git-map en de origin-URL. Een opgegeven naam moet overeenkomen met `[a-z0-9][a-z0-9-]{0,63}`. Zonder naam genereert OpenClaw `wt-`, gevolgd door acht willekeurige hexadecimale tekens.

OpenClaw maakt branch `openclaw/<name>` aan op de aangevraagde basisreferentie. Zonder basisreferentie haalt het `origin` op, gebruikt het indien beschikbaar de standaardbranch van de remote en valt het terug op lokale `HEAD` wanneer de repository offline is of geen bruikbare remote heeft.

## Genegeerde bestanden beschikbaar stellen

Voeg `.worktreeinclude` toe aan de hoofdmap van de bronrepository om geselecteerde genegeerde, niet-bijgehouden bestanden naar een nieuwe worktree te kopiëren. Het bestand gebruikt syntaxis voor gitignore-patronen, één patroon per regel, met `#`-opmerkingen:

```gitignore
.env.local
fixtures/generated/**
```

Alleen bestanden die git zowel als genegeerd als niet-bijgehouden rapporteert, komen in aanmerking. Bijgehouden bestanden zijn al via git aanwezig en worden bij deze stap nooit gekopieerd. OpenClaw overschrijft of wijzigt geen doelbestanden die al bestaan, volgt geen mappen die symbolische koppelingen zijn en behoudt de bestandsmodi van gekopieerde bestanden. Het registreert alleen paden die het daadwerkelijk aanmaakt, zodat latere wijzigingen in het manifest er niet toe kunnen leiden dat deze bestanden hun bescherming bij opschoning verliezen.

## Repository-installatie uitvoeren

Als `.openclaw/worktree-setup.sh` in de bronrepository bestaat en uitvoerbaar is, voert OpenClaw het uit met de nieuwe worktree als huidige map. Het script ontvangt:

```text
OPENCLAW_SOURCE_TREE_PATH=<source checkout>
OPENCLAW_WORKTREE_PATH=<managed worktree>
```

Een afsluitcode die niet nul is, breekt de aanmaak af en verwijdert de nieuwe worktree en branch. Dit is een repositorylokaal contract; er is geen OpenClaw-configuratiesleutel voor.

## Sessieworktrees

Start een geïsoleerde chat vanuit een map die door Git wordt beheerd met een worktreesessie: gebruik op de pagina Nieuwe sessie van de Control UI de kiezer **Plaats** om een bronmap van de Gateway te kiezen en selecteer vervolgens **Worktree** (met optioneel een basisbranch en worktreenaam). De keuze verschijnt pas nadat de Gateway heeft bevestigd dat de geselecteerde map een Git-checkout is; gewone mappen worden rechtstreeks uitgevoerd en tonen geen besturingselement voor Git-isolatie. iOS biedt dezelfde keuze via Chatacties en Android naast Nieuwe chat, wanneer de actieve agentwerkruimte door Git wordt beheerd.

Coderingsagents kunnen ook `spawn_task` aanroepen wanneer ze bevestigd vervolgwerk buiten de huidige taak ontdekken. De Control UI toont een suggestiechip zonder iets te starten, terwijl een door de Gateway ondersteunde TUI een interactieve prompt met dezelfde acties toont. Als je **Starten in worktree** selecteert, wordt een nieuwe sessiegebonden worktree vanuit het voorgestelde project aangemaakt en wordt de zelfstandige prompt als eerste beurt verzonden; als je de suggestie afwijst, blijft de repository ongewijzigd. Suggesties en hun ID's zijn tijdelijk en blijven niet behouden na een herstart van de Gateway.

OpenClaw stelt deze tools alleen beschikbaar aan operatorsessies met een bruikbare Gateway-UI. Kanaalsessies en lokale/ingesloten TUI-sessies ontvangen ze pas wanneer die oppervlakken over een overdraagbaar, getypeerd contract voor taakacties beschikken.

De resulterende beheerde worktree is eigendom van de sessie en elke agentuitvoering in die sessie gebruikt de checkout ervan. Wanneer de werkruimte een submap van een repository is, wordt de worktree verankerd in de hoofdmap van de repository en wordt de sessie uitgevoerd vanuit de overeenkomstige submap daarin. Voor het aanmaken van een sessieworktree wordt het `operator.write`-bereik van de methode gebruikt, maar repository-checkouthooks en de stap `.openclaw/worktree-setup.sh` worden alleen uitgevoerd voor `operator.admin`-aanroepers omdat ze repositorycode uitvoeren; het beschikbaar stellen via `.worktreeinclude` geldt nog steeds voor elke aanroeper. Als je de sessie verwijdert, wordt de worktree alleen verwijderd wanneer dit zonder verlies kan. Vuile worktrees of branches met niet-gepushte commits blijven beschikbaar; de uurlijkse opschoning maakt na 7 inactieve dagen snapshots van sessieworktrees, waarbij recente sessieactiviteit als worktreeactiviteit telt. Verwijderde worktrees kunnen vanuit hun snapshots worden hersteld zoals hieronder beschreven.

`sessions.create` kan een absoluut `cwd` bevatten om rechtstreeks in een andere Gateway-map uit te voeren, om samen met `worktree: true` de broncheckout te kiezen of om de werkmap van een gekoppelde Node in te stellen. Elk expliciet hostpad vereist `operator.admin`; het gewone aanmaken van worktreechats blijft `operator.write` en blijft verankerd in de geconfigureerde werkruimte.

`sessions.create` accepteert naast `worktree: true` ook `worktreeBaseRef` en `worktreeName` om de basisreferentie en de worktreenaam te kiezen (de branch wordt `openclaw/<name>`); beide blijven op `operator.write`. De aangemaakte worktree wordt geretourneerd in het aanmaakresultaat en als `worktree: { id, branch, repoRoot }` opgeslagen in de sessierij, zodat sessielijsten de checkout en branch kunnen tonen. Bij het verwijderen van een sessie wordt een behouden vuile checkout als `worktreePreserved` gemeld in plaats van deze stilzwijgend achter te laten.

## Snapshots, opschoning en herstel

Bij verwijdering wordt eerst een synthetische commit aangemaakt met bijgehouden en niet-genegeerde, niet-bijgehouden bestanden, waarna deze wordt vastgezet op `refs/openclaw/snapshots/<id>`. Genegeerde bestanden komen nooit in de objectdatabase van de repository terecht. OpenClaw slaat alleen de genegeerde bestanden die het daadwerkelijk beschikbaar heeft gesteld op in opgesplitste rijen van de gedeelde statusdatabase; de geregistreerde verzameling paden blijft gezaghebbend, zelfs als `.worktreeinclude` later verandert of verdwijnt. Bij herstel worden die bytes uit de onveranderlijke snapshot gelezen en worden hun volledige modi opnieuw toegepast. Automatische opschoning behoudt een actieve worktree wanneer van een geregistreerd pad niet langer veilig een snapshot kan worden gemaakt. Als het maken van de snapshot mislukt, stopt de verwijdering. Een expliciete geforceerde verwijdering kan zonder snapshot doorgaan.

OpenClaw past deze opschoningsregels toe:

- Aan het einde van een uitvoering verwijdert het een worktree alleen wanneer `git status --porcelain` leeg is en `git log HEAD --not --remotes --oneline` geen niet-gepushte commits vindt. Anders geeft het alleen de activiteitsvergrendeling vrij.
- De uurlijkse opschoning maakt snapshots van ontgrendelde worktrees die eigendom zijn van Workboard of een sessie en langer dan 7 dagen inactief zijn, en verwijdert ze, zelfs wanneer ze vuil zijn. Handmatige worktrees worden nooit automatisch verwijderd.
- Snapshotrecords blijven 30 dagen herstelbaar. Daarna verwijdert de opschoning de snapshotreferentie en registerrij.
- Een actieve OpenClaw-procesvergrendeling en elke vreemde of niet-herkende git-worktreevergrendeling beschermen een worktree tegen garbagecollection.

Bij herstel wordt `openclaw/<name>` opnieuw aangemaakt op de oorspronkelijke commit van vóór de snapshot, waarna de verschillen uit de snapshot opnieuw worden opgebouwd als niet-gestagede wijzigingen en niet-bijgehouden bestanden. Hierdoor blijft de synthetische snapshotcommit buiten de branchgeschiedenis. De snapshotreferentie blijft als herkomst geregistreerd.

## CLI

```bash
openclaw worktrees list [--json]
openclaw worktrees create <repo-root> [--name <name>] [--base-ref <ref>] [--json]
openclaw worktrees remove <id> [--force] [--json]
openclaw worktrees restore <id> [--json]
openclaw worktrees gc [--json]
```

De pagina **Worktrees** van de Control UI onder Instellingen biedt dezelfde acties plus aanmaak met een kiezer voor de basisbranch, toont de eigenaar van elke worktree (handmatig, Workboard of de eigenaarsessie met een koppeling naar de bijbehorende chat) en biedt de mogelijkheid om geforceerd opnieuw te proberen wanneer bij een verwijdering een mislukte snapshot wordt gemeld.

## Gateway-methoden

| Methode               | Doel                                                                 |
| -------------------- | ----------------------------------------------------------------------- |
| `worktrees.list`     | Actieve en herstelbare worktreerecords weergeven.                            |
| `worktrees.branches` | Lokale en remote-branches van een repository weergeven voor kiezers van basisreferenties.    |
| `worktrees.create`   | Een benoemde beheerde worktree aanmaken of hergebruiken.                               |
| `worktrees.remove`   | Een snapshot van een worktree maken en deze verwijderen. Geforceerde verwijderingen melden `snapshotError`. |
| `worktrees.restore`  | Een verwijderde worktree vanuit de snapshot herstellen.                           |
| `worktrees.gc`       | Opschoning van inactieve en verweesde items en bewaarperioden nu uitvoeren.                            |

`worktrees.list` vereist `operator.read` en de wijzigende methoden vereisen `operator.admin`. `worktrees.branches` heeft `operator.write` nodig voor geconfigureerde agentwerkruimten, terwijl elk ander hostpad `operator.admin` vereist (overeenkomstig de cwd-drempel van `sessions.create`). Het leest alleen bestaande referenties en haalt nooit iets op, en branches die alleen op een remote bestaan, worden met remote-kwalificatie geretourneerd (`origin/feature-a`), zodat elke geretourneerde naam als basisreferentie kan worden herleid. Nieuwe sessie kan via deze methode ook een getypeerde repositorystatus aanvragen; een gewone map of niet-beschikbare checkout retourneert geen branches in plaats van de UI te dwingen Git-ondersteuning uit een fouttekst af te leiden.

## Workboard-werkruimten

De meegeleverde [Workboard-plugin](/nl/plugins/workboard) kan een kaartwerkruimte als beheerde worktree materialiseren:

```json
{
  "kind": "worktree",
  "path": "/absolute/path/to/source-checkout",
  "branch": "main"
}
```

`path` identificeert de Git-broncheckout. `branch` is optioneel en wordt de basisreferentie. Voor een aanroeper met volledige hosttoegang maakt of hergebruikt Workboard `wb-<card-id>`, voert het de subagent uit met de beheerde checkout als werkmap en schrijft het opgeloste pad en de branch terug naar de kaart. Gateway-clients hebben `operator.admin` nodig voor materialisatie met volledige hosttoegang. Aan het einde van een uitvoering verwijdert Workboard de checkout alleen wanneer aantoonbaar geen verlies optreedt; vuil werk of niet-gepushte commits blijven beschikbaar.

Voor een werkruimtegebonden aanroeper moeten `path` en de hoofdmap van de repository exact overeenkomen met de doelwerkruimte van de agent. Workboard wordt vervolgens rechtstreeks in die map uitgevoerd en registreert een mapwerkruimte in plaats van een beheerde worktree op de host te materialiseren. Het doel moet voor dezelfde werkruimte een beschrijfbare, niet-gedeelde Docker-sandbox gebruiken, de hash van de actieve container moet overeenkomen met de aangevraagde koppelingen en het beleid, en het doel mag geen verhoogde uitvoering, hostbeheer, hostbrede sessies, permanente host-/Node-uitvoering of niet-geclassificeerde Plugin- en MCP-tools beschikbaar stellen. Als het doelbeleid of de actieve container ruimere toegang biedt, laat de verzending de kaart niet-toegewezen en meldt deze de incompatibele status.
