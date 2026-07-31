---
read_when:
    - Je moet begrijpen waarom een CI-taak wel of niet is uitgevoerd
    - Je debugt een mislukte GitHub Actions-controle
    - Je coördineert een releasevalidatieronde of een herhaling daarvan
    - Je wijzigt de ClawSweeper-dispatch of het doorsturen van GitHub-activiteit
summary: CI-taakgrafiek, scopepoorten, releaseparaplu's en lokale opdrachtequivalenten
title: CI-pijplijn
x-i18n:
    generated_at: "2026-07-27T04:48:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9de5b527354f3cc9eed3813e961116f3834c61bd72b29c92f762c46722815df
    source_path: ci.md
    workflow: 16
---

OpenClaw CI wordt uitgevoerd bij pushes naar `main` (Markdown- en `docs/**`-paden worden bij
de trigger genegeerd), bij elke pull request die geen concept is en bij handmatige uitvoering.
Canonieke pushes naar `main` worden één voor één uitgevoerd: met de gelijktijdigheidsgroep `CI` kan één
volledige integratiecyclus worden uitgevoerd, terwijl GitHub alleen de nieuwste wachtende push bewaart.
Nieuwe merges vervangen die wachtende uitvoering in plaats van werk te annuleren waarvoor al
een Blacksmith-matrix is geregistreerd. Bij pull requests worden achterhaalde heads nog steeds geannuleerd
en handmatige uitvoeringen gebruiken geïsoleerde groepen. `preflight` classificeert de diff en
schakelt kostbare lanes uit wanneer alleen niet-gerelateerde onderdelen zijn gewijzigd. Handmatige
uitvoeringen van `workflow_dispatch` omzeilen bewust slimme afbakening en waaieren uit over de
volledige graaf voor releasekandidaten en brede validatie. Android-lanes blijven
optioneel via `include_android` (of de invoer `release_gate`). Plugin-dekking die uitsluitend voor releases geldt,
bevindt zich in de afzonderlijke workflow
[`Plugin Prerelease`](#plugin-prerelease) en wordt alleen uitgevoerd vanuit
[`Full Release Validation`](#full-release-validation) of via een expliciete handmatige
uitvoering.

## Overzicht van de pijplijn

| Taak                               | Doel                                                                                                                                                                                                                  | Wanneer deze wordt uitgevoerd                       |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `preflight`                        | Gewijzigde bereiken detecteren en het CI-manifest samenstellen; bij canonieke Node-relevante `main` de momentopname van afhankelijkheden vóór het uitwaaieren vernieuwen en onderhouden                             | Altijd bij pushes en PR's die geen concept zijn     |
| `security-fast`                    | Detectie van privésleutels, controle van gewijzigde workflows via `zizmor` en controle van het productielockbestand                                                                                               | Altijd bij pushes en PR's die geen concept zijn     |
| `pnpm-store-warmup`                | De door het lockbestand vastgelegde Actions-cache opwarmen voor pull requests en handmatige uitvoeringen zonder Linux Node-shards te blokkeren                                                                          | Node- of docs-controlelanes buiten main geselecteerd |
| `build-artifacts`                  | `dist/`, Control UI, smokecontroles voor de gebouwde CLI, opstartgeheugen en ingesloten controles van gebouwde artefacten bouwen                                                                                   | Node-relevante wijzigingen                          |
| `control-ui-i18n`                  | Gegenereerde Control UI-localebundels, metagegevens en vertaalgeheugen verifiëren; adviserend bij automatische uitvoeringen, blokkerend bij handmatige release-CI                                                       | Control UI-i18n-relevante wijzigingen en handmatige CI |
| `checks-fast-core`                 | Snelle Linux-correctheidslanes: ratel voor het maximale aantal regels van de onderdrukkingsbasislijn, gebundeld + protocol, Bun-starter en de snelle taak voor CI-routering                                             | Node-relevante wijzigingen                          |
| `qa-smoke-ci-profile`              | Twee zelfstandige, evenwichtige delen van de begrensde representatieve automatische QA Smoke-set; volledige taxonomiedekking blijft beschikbaar via expliciete QA-profielen                                            | Node-relevante wijzigingen                          |
| `checks-fast-contracts-plugins-*`  | Twee gewogen shards voor Plugin-contracten                                                                                                                                                                             | Node-relevante wijzigingen                          |
| `checks-fast-contracts-channels-*` | Twee gewogen shards voor kanaalcontracten                                                                                                                                                                              | Node-relevante wijzigingen                          |
| `checks-node-*`                    | Node-tests voor gewijzigde doelen bij pull requests; volledige kernshards bij `main`, handmatige, release- en brede terugvaluitvoeringen                                                                         | Node-relevante wijzigingen                          |
| `check-*`                          | Ges hard equivalent van de lokale hoofdgate: bewakingen, shrinkwrap, configuratiemetagegevens van gebundelde kanalen, productietypen, lint, afhankelijkheden en testtypen                                             | Node-relevante wijzigingen                          |
| `check-additional-*`               | Strookgewijze grenscontroles (inclusief afwijkingen in promptmomentopnamen), grenzen voor sessietoegang/transcriptlezers/SQLite-transacties, lintgroepen voor extensies, compilatie/canary van pakketgrenzen en runtime-topologiearchitectuur | Node-relevante wijzigingen                          |
| `checks-node-compat-node22`        | Compatibiliteitsbuild en smokelane voor Node 22                                                                                                                                                                        | Handmatige CI-uitvoering voor releases               |
| `check-docs`                       | Opmaak-, lint- en controles op verbroken links voor documentatie                                                                                                                                                       | Documentatie gewijzigd (PR's en handmatige uitvoering) |
| `native-i18n`                      | Veilige extractie en lokalisatie van native broncode verifiëren bij bron-PR's; volledige pariteit van vertaalde/platformgegenereerde inhoud afdwingen bij gegenereerde PR's en handmatige CI                          | Native i18n-relevante wijzigingen                   |
| `skills-python`                    | Ruff + pytest voor door Python ondersteunde Skills                                                                                                                                                                     | Wijzigingen die relevant zijn voor Python-Skills    |
| `checks-windows`                   | Windows-specifieke proces-/padtests plus gedeelde regressies in importspecificaties van de runtime                                                                                                                     | Windows-relevante wijzigingen                       |
| `macos-node`                       | Gerichte macOS-TypeScript-tests: launchd, Homebrew, runtimepaden, pakketteringsscripts en procesgroepwrapper                                                                                                          | macOS-relevante wijzigingen                         |
| `macos-swift`                      | Swift-lint en -build voor de macOS-app, plus tests voor de app en het gedeelde OpenClawKit-pakket                                                                                                                      | macOS-relevante wijzigingen                         |
| `ios-build`                        | Generatie van het Xcode-project plus de simulatorbuild van de iOS-app                                                                                                                                                 | Wijzigingen aan de iOS-app, gedeelde app-kit of Swabble |
| `android`                          | Android-eenheidstests voor beide varianten plus één debug-APK-build                                                                                                                                                  | Android-relevante wijzigingen                       |
| `openclaw/ci-gate`                 | Eindaggregaat: vereist preflight en beveiliging; accepteert alleen overgeslagen taken voor downstream-lanes die door het manifest zijn uitgeschakeld                                                                    | Elke CI-uitvoering die geen concept is               |
| `test-performance-agent`           | Afzonderlijke workflow: dagelijkse optimalisatie van trage Codex-tests na vertrouwde activiteit                                                                                                                        | Succesvolle CI op main of handmatige uitvoering      |
| `openclaw-performance`             | Afzonderlijke workflow: dagelijkse/op aanvraag gemaakte Kova-runtimeprestatierapporten met mockprovider-, diep-profilerings- en live GPT 5.6-lanes                                                                    | Geplande en handmatige uitvoering                    |

Zelfstandige Periphery-workflows dwingen af dat er geen bevindingen voor dode code zijn in de iOS- en macOS-apps. De gedeelde OpenClawKit-workflow scant beide afnemers parallel en rapporteert een declaratie alleen wanneer Periphery vanuit beide builds dezelfde Swift USR uitvoert. Het gegenereerde `OpenClawProtocol/GatewayModels.swift`-schemacontract wordt behouden als door de generator beheerde code en niet behandeld als app-lokale dode code.

## Volgorde voor snel falen

1. `preflight` bepaalt welke lanes überhaupt bestaan. De logica van `docs-scope` en `changed-scope` bestaat uit stappen binnen deze taak, niet uit zelfstandige taken. Canonieke `main` start onmiddellijk, maar de gelijktijdigheidsgroep laat slechts één volledige uitvoering toe en voegt latere pushes samen tot één nieuwste wachtende uitvoering. Node-relevante pushes naar main serialiseren hier ook de enige schrijver naar de afhankelijkhedenschijf en het onderhoud van de omvang daarvan voordat downstream-taken de sleutel mogen koppelen; Blacksmith kan een nieuwe commit pas aan een latere workflow-uitvoering beschikbaar stellen, zodat consumenten binnen dezelfde uitvoering de door een markering gecontroleerde lokale terugval behouden.
2. `security-fast`, `check-*`, `check-additional-*`, `check-docs` en `skills-python` falen snel zonder op de zwaardere artefact- en platformmatrixtaken te wachten.
3. `build-artifacts` en de localecontroles overlappen met de snelle Linux-lanes. Bron-PR's voor Control UI en native apps sluiten gegenereerde localemomentopnamen/-resources uit; hun geserialiseerde vernieuwingsworkflows repareren en automatisch mergen geïsoleerde gegenereerde PR's op de achtergrond. Bron-CI blokkeert nog steeds verouderde broninventarissen en onveilige lokalisatieaanroepen. Gegenereerde PR's, handmatige CI en releasevoorbereiding dwingen volledige pariteit van vertaalde/platformgegenereerde inhoud af. Canonieke `release/YYYY.M.PATCH`-branches kunnen reparaties van locales voor releasevoorbereiding samen met de overige gegenereerde release-uitvoer bevatten.
4. Daarna waaieren zwaardere platform- en runtimelanes uit: `checks-fast-core`, `checks-fast-contracts-plugins-*`, `checks-fast-contracts-channels-*`, `checks-node-*`, `checks-windows`, `macos-node`, `macos-swift`, `ios-build` en `android`.
5. `openclaw/ci-gate` wacht op elke geselecteerde lane. Preflight en beveiliging moeten slagen; downstream-taken mogen alleen worden overgeslagen wanneer het manifest ze niet heeft geselecteerd. Een mislukte of geannuleerde geselecteerde lane laat het aggregaat mislukken.

De mergecoördinator mag een geverifieerde, geslaagde `openclaw/ci-gate`
voor dezelfde pull-request-head maximaal 24 uur hergebruiken. Dit voorkomt dat een
bijdragersbranch na niet-gerelateerde wijzigingen aan `main` opnieuw moet worden geschreven. Het herbruikbare resultaat
vervangt niet de afzonderlijke strikte, door de App beheerde test-mergecontrole tegen de huidige `main`.
Een latere wachtende of mislukte heruitvoering wist een eerder geslaagd resultaat voor
die ongewijzigde head tijdens het geldigheidsvenster niet.

De regelset voor de standaardbranch vereist de door GitHub Actions beheerde controle `openclaw/ci-gate`. Repositorybeheerders en -admins beschikken over een gecontroleerde noodomzeiling die uitsluitend bedoeld is voor ondertekende directe fast-forward-landingen; de regelset van de organisatie blokkeert nog steeds verwijderingen en niet-fast-forward-updates. Normale merges van pull requests moeten de controle blijven gebruiken in plaats van mislukte CI te omzeilen. De afzonderlijke strikte, door de App beheerde testmergecontrole koppelt de head nog steeds aan de huidige `main`.

GitHub kan vervangen jobs van pull requests als `cancelled` markeren wanneer een nieuwere head wordt geland. Beschouw dat als CI-ruis, tenzij de nieuwste run voor dezelfde PR ook mislukt. Canonieke `main`-runs worden na toelating niet geannuleerd; wanneer er mergeverkeer binnenkomt, vervangt GitHub alleen de oudere wachtende run door de nieuwste tip. Matrixjobs gebruiken `fail-fast: false`, en `build-artifacts` rapporteert fouten in ingebedde kanalen, de grens voor core-ondersteuning en gateway-bewaking rechtstreeks, in plaats van kleine verificatiejobs in de wachtrij te plaatsen. De automatische CI-concurrenciesleutel heeft een versie (`CI-v7-*`), zodat een zombie aan GitHub-zijde in een oude wachtrijgroep nieuwere runs op main niet onbeperkt kan blokkeren. Handmatige runs van de volledige suite gebruiken `CI-manual-v1-*` en annuleren geen lopende runs. De geheugenbewaking bij het opstarten van de pluginlijst hanteert een limiet van 350 MiB op zelfgehoste Blacksmith Linux en staat 425 MiB toe op door GitHub gehoste Linux, waarvan de RSS-basiswaarde voor dezelfde gebouwde CLI hoger is.

Gebruik `pnpm ci:timings`, `pnpm ci:timings:recent` of `node scripts/ci-run-timings.mjs <run-id>` om de verstreken tijd, wachtrijtijd, langzaamste jobs, fouten en de `pnpm-store-warmup`-fan-outbarrière uit GitHub Actions samen te vatten. De job `ci-timings-summary` binnen de workflow bestaat in `ci.yml`, maar is momenteel uitgeschakeld (`if: false`); voer in plaats daarvan lokaal de timinghelper uit. Controleer voor buildtiming de stap `Build dist` van de job `build-artifacts`: `pnpm build:ci-artifacts` toont `[build-all] phase timings:` en bevat `ui:build`; de job uploadt ook het artefact `startup-memory`.

## PR-context en bewijs

PR's van externe bijdragers voeren een controle voor PR-context en bewijs uit vanuit
`.github/workflows/real-behavior-proof.yml`. De workflow checkt de
vertrouwde workflowrevisie (`github.workflow_sha`) uit en evalueert alleen de PR-beschrijving;
deze voert geen code uit de branch van de bijdrager uit.

De controle is van toepassing op PR-auteurs die geen repository-eigenaar, lid,
medewerker of bot zijn. De controle slaagt wanneer de PR-beschrijving door de auteur geschreven
secties `What Problem This Solves` en `Evidence` bevat. Het bewijs kan een gerichte
test, een CI-resultaat, schermafbeelding, opname, terminaluitvoer, livewaarneming,
geredigeerd logboek of een link naar een artefact zijn. De beschrijving geeft de bedoeling en nuttige validatie;
reviewers inspecteren de code, tests en CI om de correctheid te beoordelen.

Werk bij een mislukte controle de PR-beschrijving bij in plaats van nog een codecommit te pushen.

## Bereik en routering

De bereiklogica staat in `scripts/ci-changed-scope.mjs` en wordt gedekt door unittests in `src/scripts/ci-changed-scope.test.ts`. Handmatige uitvoering slaat detectie van het gewijzigde bereik over en laat het preflightmanifest handelen alsof elk afgebakend gebied is gewijzigd.

Afzonderlijke Periphery-workflows voor iOS en macOS handhaven een beleid van nul bevindingen voor dode code. Elke workflow wordt alleen uitgevoerd wanneer een pull request die geen concept is het bijbehorende native scanbereik raakt, of bij handmatige uitvoering.

- **Wijzigingen aan CI-workflows** valideren de Node-CI-graaf, workflowlinting en de Windows-lane (`ci.yml` voert deze uit), maar dwingen op zichzelf geen native builds voor iOS, Android of macOS af; die platformlanes blijven beperkt tot wijzigingen in de platformbroncode.
- **Workflowcontrole** voert `actionlint`, `zizmor` voor alle workflow-YAML-bestanden, de beveiliging voor interpolatie van samengestelde acties en de beveiliging tegen conflictmarkeringen uit. De tot PR's beperkte job `security-fast` voert ook `zizmor` uit voor gewijzigde workflowbestanden, zodat beveiligingsbevindingen in workflows vroeg in de hoofdgraaf van de CI mislukken.
- **Documentatie bij pushes naar `main`** wordt gecontroleerd door de zelfstandige workflow `Docs` met dezelfde ClawHub-documentatiespiegel die door CI wordt gebruikt, zodat gemengde pushes met code en documentatie niet ook de CI-shard `check-docs` in de wachtrij plaatsen. Pull requests en handmatige CI voeren `check-docs` nog steeds vanuit CI uit wanneer documentatie is gewijzigd.
- **TUI PTY** wordt voor TUI-wijzigingen uitgevoerd in de Linux Node-shard `checks-node-core-runtime-tui-pty`. De shard voert `test/vitest/vitest.tui-pty.config.ts` uit met `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1`, zodat deze zowel de deterministische fixturelane `TuiBackend` als de langzamere smoke-test `tui --local`, die alleen het externe modeleindpunt mockt, dekt.
- **Wijzigingen die alleen CI-routering betreffen, de kleine reeks core-testfixtures die de snelle taak rechtstreeks uitvoert en beperkte wijzigingen aan helpers voor plugincontracten** gebruiken een snel manifestpad dat uitsluitend Node gebruikt: `preflight`, `security-fast` en alleen de snelle lanes waarop de wijziging betrekking heeft — één CI-routeringstaak `checks-fast-core`, de twee shards voor plugincontracten, of beide. Dat pad slaat buildartefacten, compatibiliteit met Node 22, kanaalcontracten, volledige core-shards, shards voor gebundelde plugins en aanvullende beveiligingsmatrices over.
- **Windows Node-controles** zijn beperkt tot Windows-specifieke wrappers voor processen en paden, npm/pnpm/UI-runnerhelpers, configuratie van pakketbeheerders en de CI-workflowoppervlakken die die lane uitvoeren; niet-gerelateerde wijzigingen aan broncode, plugins, installatiesmoketests en uitsluitend tests blijven op de Linux Node-lanes.

De langzaamste Node-testfamilies worden opgesplitst of uitgebalanceerd, zodat elke job klein blijft zonder te veel runners te reserveren:

- Plugincontracten en kanaalcontracten worden elk uitgevoerd als twee gewogen, door Blacksmith ondersteunde shards met de standaardfallback naar de GitHub-runner.
- Snelle/ondersteunende lanes voor core-unittests worden afzonderlijk uitgevoerd; de infrastructuur van de core-runtime wordt opgesplitst in shards voor processen, gedeelde onderdelen, hooks, secrets en drie cron-domeinen.
- Automatisch antwoorden wordt uitgevoerd met evenwichtig verdeelde workers, waarbij de antwoordsubtree wordt opgesplitst in shards voor agent-runner, opdrachten, dispatch, sessies en statusroutering.
- Configuraties voor de agentische gateway/server (control-plane) worden verdeeld over lanes voor chat, authenticatie, modellen, HTTP/plugins, runtime en opstarten, in plaats van op gebouwde artefacten te wachten.
- Normale CI verpakt alleen geïsoleerde infrastructuurshards met include-patronen in deterministische bundels van maximaal 64 testbestanden. Dit verkleint de Node-matrix zonder niet-geïsoleerde opdracht-/cron-suites, stateful agents-core-suites of gateway-/serversuites samen te voegen. Zware vaste suites blijven op 8 vCPU, terwijl de gebundelde lanes en lanes met een lager gewicht 4 vCPU gebruiken.
- Pull requests in de canonieke repository hergebruiken de resolver voor gewijzigde tests op basis van de synthetische diff van de samengevoegde tree. Nauwkeurige wijzigingen voeren één gerichte Node-job uit; elk geselecteerd testbestand krijgt een eigen proces, zodat de isolatie van stateful suites intact blijft. De planner combineert tests van siblings met afhankelijke onderdelen uit de importgrafiek en valt terug op het bestaande compacte volledige-suiteplan van 14 jobs voor wijzigingen aan workspace-pakketten, pakketten/lockfiles, de gedeelde harness, gesplitste configuratie, hernoemde of verwijderde onderdelen, openbare contractwijzigingen van extensies, tests met een speciale shardconfiguratie, gedeeltelijk opgeloste of lege doelen, te grote pad- of doelplannen en plannerfouten. Gerichte plannen behouden altijd de volledige grenscontrole voor gebouwde artefacten, omdat de repositoryscanners daarvan niet uit imports kunnen worden afgeleid. `main`-pushes voeren dezelfde volledige compacte suite uit: openstaande tussenliggende pushgebeurtenissen kunnen worden samengevoegd, dus de nieuwste overblijvende run moet de volledige integratietree valideren in plaats van alleen de uiteindelijke diff van één push. Handmatige dispatches en releasepoorten behouden de volledige benoemde matrix per shard.
- De volledige Node-matrix laat eerst de consequent trage seriële tooling, de shards voor opdrachten voor automatisch antwoorden en de brede cacheschrijver van core-fast toe. Hierdoor blijft de limiet van 28 jobs behouden en wordt voorkomen dat werk op het kritieke pad en de transformseed van de volgende run naar een latere golf verschuiven.
- Brede browser-, QA-, media- en overige plugintests gebruiken hun eigen Vitest-configuraties in plaats van de gedeelde catch-all voor plugins. Shards met include-patronen registreren timingitems met de naam van de CI-shard, zodat `.artifacts/vitest-shard-timings.json` een volledige configuratie van een gefilterde shard kan onderscheiden.
- Linux Node-shardjobs bewaren Vitests experimentele bestandssysteemcache voor modules via de upstream Actions-cache-API, die Blacksmith transparant versnelt op zijn runners. Elke CI-shard herstelt alleen en pakt de beveiligde seed uit in een eigen lokale root op de runner; de shardwrapper geeft gelijktijdige Vitest-processen vervolgens afzonderlijke actieve submappen. Alleen de niet-annuleerbare dagelijkse of expliciet gestarte warmer slaat een nieuw onveranderlijk archief op, zodat pull requests geen transformaties kunnen publiceren of cachefamilies per PR kunnen aanmaken. Een vingerafdruk van de transformatie-invoer wist incompatibele generaties van lockfiles, pakketten, tsconfig en Vitest-configuraties. De beveiligde schrijver scant en snoeit zijn herstelde cache tot 75% nadat deze groter is geworden dan 2 GiB. Vitest hasht de module-id, broninhoud, omgeving en opgeloste transformatieconfiguratie, zodat gewone gedeeltelijke bronwijzigingen ongewijzigde items warm houden terwijl gewijzigde modules veilig een cachemisser opleveren. Grove herstelprefixen overbruggen workflowruns; de normale LRU- en inactiviteitsverwijdering van de Actions-cache begrenst oude onveranderlijke archieven.
- Vertrouwde Linux Node-jobs koppelen ook de pnpm-store en `node_modules` vanuit één beveiligde afhankelijkhedenschijf per ondersteunde Node-lijn. Pakketmanifesten, installatie-instellingen, het runnerplatform en de exacte Node-patch maken geen deel uit van de schijfsleutel; een exacte vingerafdruk van de runtime en installatie-invoer bepaalt of een job de tree hergebruikt of opnieuw installeert en dezelfde schijf vernieuwt. Manifesten worden vóór het hashen gecanonicaliseerd. De gecontroleerde directe roothooks behouden alleen de installatielifecycle-scripts van pnpm, zodat wijzigingen aan formatterings- en gewone test-/buildscripts de warme afhankelijkhedentree behouden; niet-gecontroleerde afwijkingen in lifecycle-hooks worden veilig geweigerd totdat hun broninvoer deel uitmaakt van het vingerafdrukcontract. Wijzigingen aan afhankelijkheden, de pakketbeheerder, hookbronnen en het lockfile maken de snapshot altijd ongeldig. Een overeenkomende vingerafdruk is noodzakelijk maar niet voldoende: de setup controleert ook het importerarchief en de manifestchecksums en verifieert vervolgens de door postinstall behouden, uit het register afkomstige lockfile-afhankelijkheden aan de hand van de pakketmanifesten die Node vanuit hun importers oplost. Ontbrekende of verouderde importerinhoud valt terug op een nieuwe installatie in plaats van de root-hoist te leveren. Een pull request waarvan de alleen-lezen snapshot onbruikbaar is, ontkoppelt de workspace-bind en installeert in lokale opslag op de runner, waardoor trage schrijfbewerkingen naar een kloon die niet kan worden gepubliceerd worden vermeden. Sticky koude installaties schakelen de interne fetch-pogingen van pnpm uit en voeren maximaal drie begrensde volledige installatiepogingen uit vanuit de geleidelijk opgewarmde store; een time-out blijft een fout. Na een inhoudelijk gevalideerd herstel of een installatie met bevroren lockfile schakelt de setup de redundante afhankelijkheidscontrole vóór uitvoering van pnpm uit: de repository snoeit opzettelijk pluginlokale `node_modules`, die pnpm anders als verouderd beschouwt en herstelt via onveilige gelijktijdige impliciete installaties tijdens de shardfan-out. De preflight van canonieke main is de enige schrijver en meet de store bij elke vernieuwing; `pnpm store prune` wordt pas uitgevoerd nadat uitgefaseerde pakketversies de omvang boven 8 GiB hebben gebracht. De publicatie van Blacksmith-snapshots verloopt asynchroon, zelfs nadat een schrijversjob is voltooid, waardoor de eerste run na een nieuwe sleutel of vingerafdruk koud kan blijven; latere, inhoudelijk gevalideerde herstelbewerkingen met exacte markers vormen het bewijs van de uitrol. Vereiste CI-jobs en pull requests krijgen wegwerpkloons, zodat wijzigingen aan afhankelijkheden geen nieuwe schijven, concurrerende snapshots of een cachevergrendeling creëren die builds kan annuleren.
- Node-shardjobs en jobs voor buildartefacten herstellen ook Nodes overdraagbare compileercache op schijf via onveranderlijke Actions-caches. Onafhankelijke naamruimten `test` en `build` voorkomen dat hun schrijvers elkaars archieven vervangen: de geplande testwarmer beheert de beveiligde testseed, terwijl `build-artifacts` per UTC-dag maximaal één beveiligd buildarchief mag publiceren vanuit vertrouwde `main`-pushes. PR- en gewone testjobs lezen alleen beveiligde snapshots, zodat bytecode van featurebranches nooit in de gedeelde seed terechtkomt en PR-verkeer geen cachearchieven creëert. Hiermee wordt V8-bytecode hergebruikt voor door Node geladen orkestratie, buildtooling en externe afhankelijkheden tussen verschillende checkoutpaden, ook wanneer slechts een deel van de brongrafiek verandert. Vitest-childprocessen schakelen een overgeërfde compileercache uit, omdat coverage binnen dynamische configuraties kan worden ingeschakeld en V8-coverage de nauwkeurigheid van bronposities kan verliezen wanneer scripts vanuit bytecode worden gedeserialiseerd.
- De job voor buildartefacten bewaart ook op inhoudsvingerafdrukken gebaseerde uitvoer van `build-all`-stappen. De door CI zelf gebouwde declaraties van de Plugin-SDK hashen de volledige TypeScript-/JSON-brongrafiek die eigendom is van de repository, sluiten geïnstalleerde en gegenereerde mappen uit en herstellen zowel platte declaraties als pakketbruggen nadat `tsdown` `dist` heeft gewist. Documentatie-, workflow-, plugin- en andere wijzigingen buiten die grafiek kunnen de declaratiesnapshot hergebruiken; bronwijzigingen bouwen deze opnieuw voordat de exportpoort wordt uitgevoerd.
- Volledige declaratiebuilds splitsen `tsdown` op in AI-, workspacepakket- en uniforme groepen. Elke groep cachet alleen declaraties en bouwt vervolgens nog steeds runtime-JavaScript opnieuw voordat die declaraties worden hersteld. Wijzigingen aan core of plugins maken daardoor alleen de grote uniforme grafiek ongeldig, terwijl wijzigingen aan workspacepakketten conservatief elke afhankelijke declaratiegroep ongeldig maken. Openbare volledige builds gebruiken doorgaans een onveranderlijke Actions-cache; grove herstelsleutels voorzien gedeeltelijke wijzigingen van een seed, inhoudsvingerafdrukken per groep wijzen verouderde gegevens af en GitHubs cachequotum verwijdert oude generaties. De wekelijkse Node 22-lane publiceert in plaats daarvan na geslaagde `main`-runs een artefact met een bewaartermijn van 14 dagen en herstelt alleen artefacten waarvan de onveranderlijke producentidentiteit op `main` naar die workflow verwijst. Dit voorkomt quotumverloop zonder toe te staan dat PR-code naar een gedeelde cache schrijft. Declaraties van Private-QA worden nooit in Actions-caches bewaard, omdat cachenaamruimten geen vertrouwelijkheidsgrenzen zijn.
- `check-additional-*` verdeelt de aanvullende lijst met grenscontroles (`scripts/run-additional-boundary-checks.mjs`) in één promptintensieve shard (`check-additional-boundaries-a`, die de controle op afwijkingen in Codex-promptsnapshots bevat) en één gecombineerde shard voor de resterende stroken (`check-additional-boundaries-bcd`). Elke shard voert onafhankelijke controles gelijktijdig uit en drukt timings per controle af. Compileer-/canarywerk voor pakketgrenzen blijft bij elkaar en de runtime-topologiearchitectuur wordt afzonderlijk uitgevoerd van de Gateway-watchcoverage die in `build-artifacts` is ingebed.
- Op de zelfgehoste buildrunner met 32 vCPU starten Gateway-watch, kanaaltests en de shard voor de ondersteuningsgrens van core samen binnen `build-artifacts`, nadat `dist/` en `dist-runtime/` al zijn gebouwd. Fallbackruns op door GitHub gehoste runners houden Gateway-watch serieel, zodat concurrentie om weinig cores de gereedheidsdeadline niet kan verbruiken.

Na toelating staat canonieke Linux-CI maximaal 28 gelijktijdige Node-testjobs toe en
12 voor de kleinere snelle/controlelanes; Windows en Android blijven op twee omdat
die runnerpools beperkter zijn. Compacte batches met volledige configuraties worden uitgevoerd met een
batchtime-out van 120 minuten, terwijl groepen met include-patronen hetzelfde begrensde
jobbudget delen.

Android-CI voert zowel `testPlayDebugUnitTest` als `testThirdPartyDebugUnitTest` uit en bouwt daarna de Play-debug-APK. De externe variant heeft geen afzonderlijke sourceset of manifest; de unittests-lane compileert de variant nog steeds met de SMS-/oproepenlogboek-BuildConfig-flags, maar voorkomt een dubbele verpakkingstaak voor de debug-APK bij elke Android-relevante push. Elke huidige Gradle-taak heeft één beveiligde sticky schijf; PR-jobs gebruiken wegwerpkloons, terwijl beveiligde runs inhoudsgeadresseerde Gradle-items ter plaatse vernieuwen.

Sleutels voor sticky schijven van Blacksmith worden bewust begrensd door ondersteunde runtime- of taakdimensies, nooit door PR-nummer, commit, run, branch of afhankelijkheidshash. Runtime-transformatie- en compileercaches gebruiken Actions-cache in plaats van sticky schijven, omdat onveranderlijke archieven verifieerbare herstel-/opslagresultaten bieden en fouten bij de promotie van veranderlijke snapshots voorkomen. Voeg na een migratie van een sticky-sleutelversie alleen de exacte verouderde sleutel-, architectuur- en regio-identiteiten toe aan `.github/retired-sticky-disks.json`, start `Sticky Disk Cleanup` vanuit `main` met dezelfde dimensies en bevestiging, verifieer de verwijdering en verwijder daarna die items. De workflow routeert ARM-identiteiten naar een ARM-runner, wijst afwijkingen in runnerregio's af, gebruikt Blacksmiths verwijderactie voor exacte sleutels en verwijdert nooit caches van Docker-builders of wildcardprefixen. Actions-cachearchieven gebruiken normale LRU- en inactiviteitsverwijdering.

De `check-dependencies`-shard voert Knip-controles voor productieafhankelijkheden, ongebruikte bestanden en ongebruikte exports uit. De controle voor ongebruikte bestanden faalt wanneer een PR een nieuw, niet-beoordeeld ongebruikt bestand toevoegt of een verouderd item in de allowlist laat staan, terwijl opzettelijke dynamische plugin-, gegenereerde, build-, livetest- en pakketbrugoppervlakken die Knip niet statisch kan oplossen behouden blijven. De controle voor ongebruikte exports sluit testondersteuningsbestanden uit en faalt bij elke ongebruikte productie-export; opzettelijke dynamische consumers moeten in `config/knip.config.ts` worden gemodelleerd. Historische doelen voeren de exportcontrole uit wanneer ze deze aanbieden en behouden anders hun oudere fallback voor dode code.

## Activiteit van ClawSweeper doorsturen

`.github/workflows/clawsweeper-dispatch.yml` is de brug aan de doelzijde van activiteit in de OpenClaw-repository naar ClawSweeper. Deze checkt geen niet-vertrouwde code van pull requests uit en voert die ook niet uit. De workflow maakt een GitHub App-token aan vanuit `CLAWSWEEPER_APP_PRIVATE_KEY` en verzendt vervolgens compacte `repository_dispatch`-payloads naar `openclaw/clawsweeper`.

De workflow heeft vier paden:

- `clawsweeper_item` voor exacte beoordelingsverzoeken voor issues en pull requests;
- `clawsweeper_comment` voor expliciete ClawSweeper-opdrachten in issue-opmerkingen;
- `clawsweeper_commit_review` voor beoordelingsverzoeken op commitniveau bij `main`-pushes;
- `github_activity` voor algemene GitHub-activiteit die de ClawSweeper-agent kan inspecteren.

Het `github_activity`-pad stuurt alleen genormaliseerde metadata door: gebeurtenistype, actie, actor, repository, itemnummer, URL, titel, status en, indien aanwezig, korte fragmenten van opmerkingen of beoordelingen. Het stuurt bewust niet de volledige webhookbody door. De ontvangende workflow in `openclaw/clawsweeper` is `.github/workflows/github-activity.yml`, die de genormaliseerde gebeurtenis naar de OpenClaw Gateway-hook voor de ClawSweeper-agent verzendt.

Algemene activiteit dient ter observatie en wordt niet standaard afgeleverd. De ClawSweeper-agent ontvangt het Discord-doel in zijn prompt en hoort alleen naar `#clawsweeper` te posten wanneer de gebeurtenis verrassend, uitvoerbaar, riskant of operationeel nuttig is. Routinematig openen en bewerken, botactiviteit, dubbele webhookruis en normaal beoordelingsverkeer horen te resulteren in `NO_REPLY`.

Behandel GitHub-titels, opmerkingen, bodies, beoordelingstekst, branchnamen en commitberichten in dit hele pad als niet-vertrouwde gegevens. Ze dienen als invoer voor samenvatting en triage, niet als instructies voor de workflow of de runtime van de agent.

## Handmatige uitvoeringen

Handmatige CI-uitvoeringen gebruiken dezelfde taakgrafiek als normale CI, maar schakelen elk afgebakend niet-Android-pad verplicht in: Linux Node-shards, shards voor gebundelde plugins, contractshards voor plugins en kanalen, compatibiliteit met Node 22, `check-*`, `check-additional-*`, smokecontroles van gebouwde artefacten, documentatiecontroles, Python-Skills, Windows, macOS, iOS-build en i18n voor de Control UI en native apps. Automatische bron-PR's verifiëren de inventaris voor native extractie en de veiligheid van Android-/Apple-lokalisatie zonder vertaalde of door het platform gegenereerde uitvoer in dezelfde PR te vereisen. De geserialiseerde workflow Native App Locale Refresh bouwt die artefacten opnieuw op in één geïsoleerde PR en schakelt automatisch samenvoegen op de exacte HEAD in nadat de vereiste controles zijn geslaagd. Volledige pariteit van native apps blijft blokkerend voor PR's met gegenereerde artefacten, handmatige CI, Full Release Validation en releasevoorbereiding. Pariteit van Control UI-landinstellingen blijft adviserend bij automatische PR- en `main`-uitvoeringen en blokkerend bij handmatige/release-CI. Zelfstandige handmatige CI-uitvoeringen voeren Android alleen uit met `include_android=true` (de invoer `release_gate` dwingt Android ook af); de overkoepelende volledige release schakelt Android in door `include_android=true` door te geven. Statische prereleasecontroles voor plugins, de alleen voor releases bestemde `agentic-plugins`-shard, de volledige batchcontrole van extensies en Docker-paden voor plugin-prereleases zijn uitgesloten van CI. De Docker-prereleasesuite wordt alleen uitgevoerd wanneer `Full Release Validation` de afzonderlijke workflow `Plugin Prerelease` start met de releasevalidatiegate ingeschakeld.

Controles van het maximale aantal PR-regels leiden de basislijn af uit de uitgecheckte synthetische mergeboom en verifiëren de bovenliggende commit van de HEAD aan de hand van de HEAD uit de gebeurtenis. Handmatige uitvoeringen gebruiken een unieke concurrencygroep, zodat een volledige suite voor een release candidate niet wordt geannuleerd door een andere push- of PR-uitvoering op dezelfde ref. Met de optionele invoer `target_ref` kan een vertrouwde aanroeper die grafiek uitvoeren voor een branch, tag of volledige commit-SHA, terwijl het workflowbestand van de geselecteerde dispatch-ref wordt gebruikt; de basislijn voor het maximale aantal regels wordt vergeleken met de merge-base van het doel en de HEAD van de standaardbranch die voor die uitvoering is vastgesteld. De invoer `release_gate` is een beheerdersfallback op basis van een exacte SHA voor PR-CI die door capaciteitsproblemen vastloopt: deze vereist dat `target_ref` een volledige commit-SHA is die overeenkomt met de HEAD van de gestarte branch en dat `pull_request_number` de open PR identificeert waarvan de mergeboom wordt gevalideerd.

```bash
gh workflow run ci.yml --ref release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=<branch-or-sha> -f include_android=true
gh workflow run full-release-validation.yml --ref main -f ref=<branch-or-sha>
```

Uitvoeringen van Gateway extended-stable voeren de npm-preflight, Full Release Validation en de
npm-release van plugins uit vanuit `extended-stable/YYYY.M.33`; de publicatie van de kern gebruikt die drie
uitvoerings-ID's plus de validatiepoging. Bewijs voor `release-ci/*` is ongeldig omdat
de publicatie elke uitvoering aan de canonieke branch en release-SHA koppelt. De tag
publiceert Gateway-images en alleen de `extended-stable*`-aliassen; dit pad slaat
de reguliere orchestrator en de bijbehorende oppervlakken voor ClawHub, native apps, GitHub Release, de website
en private dist-tags over. Zie [Maandelijkse extended-stable-publicatie van Gateway](/nl/reference/RELEASING#monthly-gateway-extended-stable-publication)
voor opdrachten en herstel.

## Runners

| Runner                          | Taken                                                                                                                                                                                                                                                                              |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ubuntu-24.04`                  | `security-fast`, handmatige CI-uitvoeringen en fallbacks voor niet-canonieke repositories, de QA Smoke-aggregatie, CodeQL-beveiligings- en kwaliteitscontroles, workflowvalidatie, labeler, automatische antwoorden, de zelfstandige Docs-workflow en de volledige Install Smoke-workflow                                |
| `blacksmith-4vcpu-ubuntu-2404`  | `preflight`, `pnpm-store-warmup`, `native-i18n`, `checks-fast-core` behalve QA Smoke-CI, contractshards voor plugins/kanalen, de meeste gebundelde/lichtere Linux Node-shards, `check-*`-paden behalve `check-lint`, geselecteerde `check-additional-*`-shards, `check-docs` en `skills-python` |
| `blacksmith-8vcpu-ubuntu-2404`  | Behouden zware Linux Node-suites, grens-/extensiezware `check-additional-*`-shards en `android`                                                                                                                                                                             |
| `blacksmith-16vcpu-ubuntu-2404` | Automatische QA Smoke-CI-shards, `build-artifacts` in CI en Testbox, en `check-lint` (voldoende CPU-gevoelig dat 8 vCPU meer kostten dan ze bespaarden)                                                                                                                                  |
| `blacksmith-8vcpu-windows-2025` | `checks-windows`                                                                                                                                                                                                                                                                  |
| `blacksmith-6vcpu-macos-15`     | `macos-node` op `openclaw/openclaw`; forks vallen terug op `macos-15`                                                                                                                                                                                                                |
| `blacksmith-12vcpu-macos-26`    | `macos-swift` en `ios-build` op `openclaw/openclaw`; forks vallen terug op `macos-26`                                                                                                                                                                                               |

## Registratiebudget voor runners

De huidige GitHub-bucket van OpenClaw voor runnerregistraties rapporteert 10.000 zelfgehoste
runnerregistraties per 5 minuten in `ghx api rate_limit`. Controleer
`actions_runner_registration` opnieuw vóór elke afstemmingsronde, omdat GitHub
deze bucket kan wijzigen. De limiet wordt gedeeld door alle Blacksmith-runnerregistraties in de
organisatie `openclaw`, dus het toevoegen van nog een Blacksmith-installatie levert geen
nieuwe bucket op.

Behandel Blacksmith-labels als de schaarse hulpbron voor piekbeheersing. Taken die
alleen routeren, meldingen versturen, samenvatten, shards selecteren of korte CodeQL-controles uitvoeren, horen
op door GitHub gehoste runners te blijven, tenzij daarvoor gemeten Blacksmith-specifieke
behoeften bestaan. Elke nieuwe Blacksmith-matrix, grotere `max-parallel` of hoogfrequente
workflow moet het maximale aantal registraties in het slechtste geval tonen en het doel op organisatieniveau
onder ongeveer 60% van de actuele bucket houden. Met de huidige bucket van 10.000 registraties
betekent dit een operationeel doel van 6.000 registraties, zodat er ruimte overblijft voor
gelijktijdige repositories, nieuwe pogingen en overlappende pieken.

Het PR-plan voor gewijzigde doelen vermindert de gebruikelijke piek van Node-tests van 14 Blacksmith-registraties tot één. PR's met een breed risico behouden de compacte fallback met 14 registraties, zodat het slechtste geval niet toeneemt.

CI voor de canonieke repository behoudt Blacksmith als het standaardrunnerpad voor normale push- en pull-requestuitvoeringen. `workflow_dispatch` en uitvoeringen voor niet-canonieke repositories gebruiken door GitHub gehoste runners, maar normale canonieke uitvoeringen controleren momenteel niet de wachtrijstatus van Blacksmith en vallen niet automatisch terug op door GitHub gehoste labels wanneer Blacksmith niet beschikbaar is.

## Ratchets voor oppervlakken

Twee budgetten die alleen mogen krimpen bewaken het configuratieoppervlak. Beide laten CI bij groei mislukken
totdat het budgetbestand bewust in dezelfde PR wordt bijgewerkt, en beide vereisen
dat het budget wordt verlaagd wanneer opschoning het werkelijke aantal verlaagt.

- `config/env-var-count-budget.txt` beperkt het aantal afzonderlijke `OPENCLAW_*`-
  namen in productiebroncode onder `src/`, `packages/` en `extensions/`
  (tests en QA Lab uitgesloten). Gecontroleerd door `node scripts/check-env-var-count.mjs`.
  Omgevingsvariabelen verwijderen: verlaag het aantal in dezelfde PR. Het toevoegen ervan is een
  beslissing over het configuratieoppervlak — motiveer die in de PR-body.
- `docs/.generated/config-baseline.counts.json` beperkt per soort
  (kern/kanaal/plugin) het aantal `openclaw.json`-schema-items. Gecontroleerd door
  `pnpm config:docs:check`; genereer opnieuw met `pnpm config:docs:gen` na elke
  schemawijziging.

## Lokale equivalenten

```bash
pnpm changed:lanes                            # inspecteer de lokale classifier voor gewijzigde lanes voor origin/main...HEAD
pnpm check:changed                            # slimme lokale controlepoort: gewijzigde opmaak/typecheck/lint/guards per begrenzingslane
pnpm check                                    # snelle lokale poort: productie-tsgo + gesharde lint + parallelle snelle guards
pnpm check:test-types
pnpm check:timed                              # dezelfde poort met timing per fase
pnpm build:strict-smoke
pnpm check:architecture
pnpm test:gateway:watch-regression
OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1 node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts
pnpm test                                     # Vitest-tests
pnpm test:changed                             # goedkope, slim gekozen gewijzigde Vitest-doelen
pnpm test:ui                                  # unit-/browsersuite voor de Control UI
pnpm ui:i18n:check                            # gegenereerde pariteit van Control UI-lokalisaties (releasepoort)
pnpm native:i18n:baseline                     # werk de door de bron beheerde inventaris van native-extracties bij
pnpm native:i18n:verify                       # broninventaris + lokalisatieveiligheid voor Android/Apple
pnpm native:i18n:check                        # strikte pariteit van vertaalde/platformgegenereerde inhoud (releasepoort)
pnpm test:channels
pnpm test:contracts:channels
pnpm check:docs                               # documentopmaak + lint + gebroken links
pnpm build                                    # bouw dist wanneer CI-artifact-/smokecontroles van belang zijn
pnpm ios:build                                # genereer en bouw het iOS-appproject
pnpm ci:timings                               # vat de nieuwste CI-run van een push naar origin/main samen
pnpm ci:timings:recent                        # vergelijk recente geslaagde CI-runs op main
node scripts/ci-run-timings.mjs <run-id>      # vat doorlooptijd, wachtrijtijd en traagste jobs samen
node scripts/ci-run-timings.mjs --latest-main # negeer ruis van issues/reacties en kies de CI van een push naar origin/main
node scripts/ci-run-timings.mjs --recent 10   # vergelijk recente geslaagde CI-runs op main
pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json
pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json
pnpm test:startup:memory
pnpm test:extensions:memory -- --json .artifacts/openclaw-performance/source/mock-provider/extension-memory.json
pnpm perf:kova:summary --report .artifacts/kova/reports/mock-provider/report.json --output .artifacts/kova/summary.md
```

## OpenClaw-prestaties

`OpenClaw Performance` is de workflow voor product-/runtimeprestaties. Deze wordt dagelijks uitgevoerd op `main` en kan handmatig worden gestart:

```bash
gh workflow run openclaw-performance.yml --ref main -f profile=diagnostic -f repeat=3
gh workflow run openclaw-performance.yml --ref main -f profile=smoke -f repeat=1 -f deep_profile=true -f live_openai_candidate=true
gh workflow run openclaw-performance.yml --ref main -f target_ref=v2026.5.2 -f profile=diagnostic -f repeat=3
```

Een handmatige start benchmarkt normaal gesproken de workflow-ref. Stel `target_ref` in om een releasetag of een andere branch te benchmarken met de huidige workflowimplementatie. Gepubliceerde rapportpaden en verwijzingen naar de nieuwste versie zijn gegroepeerd op de geteste ref, en elke `index.md` legt de geteste ref/SHA, workflow-ref/SHA, Kova-ref, het profiel, de authenticatiemodus van de lane, het model, het aantal herhalingen en de scenariofilters vast.

De workflow installeert OCM vanuit een vastgezette release en Kova vanuit `openclaw/Kova` met de vastgezette invoer `kova_ref`, en voert vervolgens drie lanes uit:

- `mock-provider`: diagnostische Kova-scenario's tegen een lokaal gebouwde runtime met deterministische nepverificatie die compatibel is met OpenAI.
- `mock-deep-profile`: CPU-/heap-/traceprofilering voor knelpunten bij het opstarten, in de Gateway en tijdens agentbeurten. Wordt volgens planning uitgevoerd, of bij handmatige start met `deep_profile=true`.
- `live-openai-candidate`: een echte OpenAI `openai/gpt-5.6-luna`-agentbeurt, overgeslagen wanneer `OPENAI_API_KEY` niet beschikbaar is. Wordt volgens planning uitgevoerd, of bij handmatige start met `live_openai_candidate=true`.

De mock-provider-lane voert na de Kova-doorgang ook broneigen OpenClaw-probes uit: opstarttijd en geheugengebruik van de Gateway voor de standaard-, overgeslagen-kanaal-, interne-hook- en opstartscenario's met vijftig plugins; RSS bij import van gebundelde plugins, herhaalde hallo-lussen van mock-OpenAI `channel-chat-baseline`, CLI-opstartopdrachten tegen de opgestarte Gateway en de SQLite-prestatiesmokeprobe voor statusgegevens. Wanneer het vorige gepubliceerde mock-provider-bronrapport voor de geteste ref beschikbaar is, vergelijkt de bronsamenvatting de huidige RSS- en heapwaarden met die basislijn en markeert deze grote RSS-stijgingen als `watch`. De Markdown-samenvatting van de bronprobe staat in `source/index.md` in de rapportbundel, met de onbewerkte JSON ernaast.

Elke lane uploadt het volledige GitHub-artifact, inclusief CPU-, heap-, trace- en gecomprimeerde diagnosebundels. Een afzonderlijke publicatiejob downloadt en valideert die artefacten, maakt vervolgens een kortlevend ClawSweeper GitHub App-token dat uitsluitend toegang heeft tot de inhoud van `openclaw/clawgrit-reports`, en geeft dit alleen door aan de Git-pushstap. Deze commit `report.json`, `report.md`, `index.md`, bronprobe-artefacten en bundelmetadata/-controlesommen onder `openclaw-performance/<tested-ref>/<run-id>-<attempt>/<lane>/`; het volledige diagnosearchief blijft in het gekoppelde Actions-artifact. De publicatiejob weigert elk rapportbestand groter dan 50 MB voordat een push wordt geprobeerd. De huidige verwijzing voor de geteste ref is `openclaw-performance/<tested-ref>/latest-<lane>.json`. Geplande runs en `profile=release`-starts mislukken als het maken van het app-token of het publiceren van het rapport mislukt. Bij handmatige niet-releasestarts blijft publicatie adviserend en blijven de GitHub-artefacten behouden wanneer authenticatie of publicatie mislukt. De vorige bronbasislijn wordt anoniem opgehaald uit de openbare rapportrepository, dus een geslaagde ophaling van de basislijn bewijst niet dat de publicatiejob is geauthenticeerd.

## Volledige releasevalidatie

`Full Release Validation` is de handmatige overkoepelende workflow voor „alles uitvoeren vóór de release”. Deze accepteert een branch, tag of volledige commit-SHA, start de handmatige workflow `CI` met dat doel (inclusief Android), start `Plugin Prerelease` voor uitsluitend releasegerelateerd bewijs voor plugins/pakketten/statische controles/Docker, start `OpenClaw Performance` tegen de doel-SHA en start `OpenClaw Release Checks` voor installatiesmoke, pakketacceptatie, pakketcontroles op verschillende besturingssystemen, QA Lab-pariteit, Matrix, Telegram en afgeschermde lanes voor Discord, WhatsApp en Slack (adviserende weergave van de volwassenheidsscorekaart is optioneel via `run_maturity_scorecard`). Stabiele en volledige profielen bevatten altijd uitgebreide live-/E2E- en soakdekking voor het Docker-releasepad; het bètaprofiel kan dit inschakelen met `run_release_soak=true`. De canonieke Telegram-E2E voor pakketten wordt binnen Pakketacceptatie uitgevoerd, zodat een volledige kandidaat geen dubbele live-poller start. Geef na publicatie `release_package_spec` door om het uitgebrachte npm-pakket opnieuw te gebruiken voor releasecontroles, Pakketacceptatie, Docker, verschillende besturingssystemen en Telegram zonder opnieuw te bouwen. Gebruik `npm_telegram_package_spec` alleen voor een gerichte heruitvoering van Telegram met een gepubliceerd pakket. De live pakket-lane van de Codex-plugin gebruikt standaard dezelfde geselecteerde status: gepubliceerd `release_package_spec=openclaw@<tag>` leidt `codex_plugin_spec=npm:@openclaw/codex@<tag>` af, terwijl SHA-/artifact-runs `extensions/codex` verpakken vanuit de geselecteerde ref. Stel `codex_plugin_spec` expliciet in voor aangepaste pluginbronnen, zoals de specificaties `npm:`, `npm-pack:` of `git:`. Het live-agentbewijs stuurt zichtbare voortgang, gaat door met willekeurige leesacties in de werkruimte en het exact schrijven van een artifact, en stuurt vervolgens een voltooiingsbericht.

Zie [Volledige releasevalidatie](/nl/reference/full-release-validation) voor de
fasematrix, exacte namen van workflowjobs, profielverschillen, artefacten en
handvatten voor gerichte heruitvoeringen.

`OpenClaw Release Publish` is de handmatige muterende releaseworkflow. Start
reguliere bèta- en stabiele publicaties vanuit een vertrouwde `main` nadat de releasetag
bestaat en nadat de OpenClaw npm-preflight is geslaagd (de preflight voert onder
zijn controles `pnpm plugins:sync:check` uit). De tag selecteert nog steeds de exacte
releasecommit, inclusief een commit op `release/YYYY.M.PATCH`; Tideclaw-alpha-
publicaties blijven hun overeenkomende alphabranch gebruiken. Hiervoor zijn de opgeslagen
`preflight_run_id` en een geslaagde
`full_release_validation_run_id` met de exacte
`full_release_validation_run_attempt` vereist, wordt `Plugin NPM Release` gestart voor alle
publiceerbare pluginpakketten, wordt `Plugin ClawHub Release` gestart voor dezelfde
release-SHA en pas daarna wordt `OpenClaw NPM Release` gestart. Voor stabiele publicatie
is ook een exacte `windows_node_tag` vereist; de workflow verifieert de Windows-bronrelease
en vergelijkt de x64-/ARM64-installatieprogramma's daarvan met de door de kandidaat goedgekeurde
invoer `windows_node_installer_digests` vóór elke publicatiesubworkflow, en promoveert en
verifieert vervolgens diezelfde vastgezette installatieprogrammadigests plus het exacte bijbehorende artifact-
en controlesomcontract voordat het GitHub-releaseconcept wordt gepubliceerd.
Voor gerichte reparaties van uitsluitend plugins gebruik je `plugin_publish_scope=selected` met een niet-lege
pakketlijst. Plugin-only `all-publishable`-runs vereisen hetzelfde onveranderlijke npm-
preflight- en Volledige-releasevalidatiebewijs als een kernpublicatie.

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Gebruik voor bewijs van een vastgezette commit op een snel veranderende branch de helper in plaats van
`gh workflow run ... --ref main -f ref=<sha>`:

```bash
pnpm ci:full-release --sha <full-sha>
```

Refs voor het starten van GitHub-workflows moeten branches of tags zijn, geen onbewerkte commit-SHA's. De
helper pusht een tijdelijke `release-ci/<sha>-...`-branch op een vertrouwde `main`-
workflow-SHA, geeft de aangevraagde doel-SHA door via de workflowinvoer `ref`,
hergebruikt strikt bewijs voor het exacte doel wanneer dit beschikbaar is, verifieert dat voor elke onderliggende
workflow `headSha` overeenkomt met de vertrouwde workflow-SHA en verwijdert de tijdelijke
branch wanneer de run is voltooid. Geef `-f reuse_evidence=false` door om nieuwe
validatie af te dwingen. De overkoepelende verificatie mislukt ook als een onderliggende workflow met een
andere workflow-SHA is uitgevoerd.

`release_profile` bepaalt de breedte van live-/providerdekking die aan releasecontroles wordt doorgegeven. De
handmatige releaseworkflows gebruiken standaard `stable`; gebruik `full` alleen wanneer je
bewust de brede adviserende provider-/mediamatrix wilt. Stabiele en volledige
releasecontroles voeren altijd de uitgebreide live-/E2E- en soaktest voor het Docker-releasepad uit;
het bètaprofiel kan dit inschakelen met `run_release_soak=true`.

- `beta` behoudt de snelste releasekritieke lanes voor OpenAI/de kern.
- `stable` voegt de stabiele provider-/backendset toe.
- `full` voert de brede adviserende provider-/mediamatrix uit.

De overkoepelende workflow legt de run-id's van de gestarte onderliggende workflows vast, en de uiteindelijke job `Verify full validation` controleert opnieuw de huidige conclusies van de onderliggende runs en voegt voor elke onderliggende run tabellen met de traagste jobs toe. Als een onderliggende workflow opnieuw wordt uitgevoerd en groen wordt, voer je alleen de bovenliggende verificatiejob opnieuw uit om het overkoepelende resultaat en de timingsamenvatting te vernieuwen.

Voor herstel accepteren zowel `Full Release Validation` als `OpenClaw Release Checks` `rerun_group`. Gebruik `all` voor een release candidate, `ci` voor alleen het normale volledige CI-subproces, `plugin-prerelease` voor alleen het subproces voor de Plugin-prerelease, `performance` voor alleen het subproces voor OpenClaw Performance, `release-checks` voor elk releasesubproces, of een specifiekere groep: `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` of `npm-telegram` in de overkoepelende workflow. Zo blijft het opnieuw uitvoeren van een mislukte releasebox beperkt na een gerichte oplossing. Combineer voor één mislukte cross-OS-lane `rerun_group=cross-os` met `cross_os_suite_filter`, bijvoorbeeld `windows/packaged-upgrade`; langdurige cross-OS-opdrachten genereren Heartbeat-regels en samenvattingen van verpakte upgrades bevatten tijdmetingen per fase. Geselecteerde QA-lanes voor Matrix en Telegram blokkeren de normale releasevalidatie, net als de dekkingspoort voor tools van het kernruntimepaar. QA-pariteit, runtimepariteit en de afgeschermde live-lanes voor Discord, WhatsApp en Slack zijn adviserend.

`OpenClaw Release Checks` gebruikt de vertrouwde workflowreferentie om de geselecteerde referentie eenmaal om te zetten in een `release-package-under-test`-tarball en geeft dat artefact vervolgens door aan cross-OS-controles en Package Acceptance, plus aan de Docker-workflow voor het live/E2E-releasepad wanneer duurtestdekking wordt uitgevoerd. Hierdoor blijven de pakketbytes consistent tussen releaseboxen en wordt voorkomen dat dezelfde kandidaat in meerdere subtaken opnieuw wordt verpakt. Voor de live-lane van de Codex-npm-Plugin geven releasecontroles een overeenkomende gepubliceerde Pluginspecificatie door die van `release_package_spec` is afgeleid, geven ze de door de operator opgegeven `codex_plugin_spec` door, of laten ze de invoer leeg zodat het Docker-script de Codex-Plugin van de geselecteerde checkout verpakt.

Dubbele uitvoeringen van `Full Release Validation` voor `ref=main` en `rerun_group=all`
vervangen de oudere overkoepelende workflow. De bovenliggende monitor annuleert elke onderliggende workflow die
al is gestart wanneer de bovenliggende workflow wordt geannuleerd, zodat nieuwere validatie van main
niet achter een verouderde releasecontrole van twee uur blijft wachten. Validatie van releasebranches/-tags
en gerichte groepen voor opnieuw uitvoeren behouden `cancel-in-progress: false`.

## Live- en E2E-shards

Het onderliggende live/E2E-releaseproces behoudt brede native `pnpm test:live`-dekking, maar voert die via `scripts/test-live-shard.mjs` uit als benoemde shards in plaats van als één seriële taak:

- `native-live-src-agents` en `native-live-src-agents-zai-coding`
- `native-live-src-gateway-core`
- op provider gefilterde `native-live-src-gateway-profiles`-taken
- `native-live-src-gateway-backends`
- `native-live-src-infra`
- `native-live-test`
- `native-live-extensions-a-k`
- `native-live-extensions-l-n`
- `native-live-extensions-moonshot`
- `native-live-extensions-openai`
- `native-live-extensions-o-z-other`
- `native-live-extensions-xai`
- gesplitste shards voor media-audio/-video en op provider gefilterde muziekshards

Dit behoudt dezelfde bestandsdekking en maakt het gemakkelijker om fouten bij trage live-providers opnieuw uit te voeren en te diagnosticeren. De samengevoegde shardnamen `native-live-src-gateway`, `native-live-extensions-o-z`, `native-live-extensions-media` en `native-live-extensions-media-music` blijven geldig voor handmatige eenmalige heruitvoeringen.

De native shards voor live-media worden uitgevoerd in `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`, gebouwd door de workflow `Live Media Runner Image`. In die image zijn `ffmpeg` en `ffprobe` vooraf geïnstalleerd; mediataken controleren vóór de configuratie alleen de binaire bestanden. Houd door Docker ondersteunde livesuites op normale Blacksmith-runners — containertaken zijn niet geschikt voor het starten van geneste Docker-tests.

Door Docker ondersteunde shards voor live-modellen/-backends gebruiken per geselecteerde commit een afzonderlijke gedeelde `ghcr.io/openclaw/openclaw-live-test:<sha>-<extensions>`-image. De live-releaseworkflow bouwt en pusht die image eenmaal, waarna de shards voor het live-Docker-model, de per provider gesharde Gateway, de CLI-backend, ACP-bind en het Codex-harnas met `OPENCLAW_SKIP_DOCKER_BUILD=1` worden uitgevoerd. Docker-shards voor de Gateway hebben expliciete limieten op scriptniveau via `timeout`, die onder de time-out van de workflowtaak liggen, zodat een vastgelopen container of opschoningspad snel mislukt in plaats van het volledige budget voor releasecontroles te verbruiken. Als die shards het volledige Docker-doel voor de broncode afzonderlijk opnieuw bouwen, is de release-uitvoering verkeerd geconfigureerd en wordt door dubbele imagebuilds doorlooptijd verspild.

## Package Acceptance

Gebruik `Package Acceptance` wanneer de vraag luidt: "werkt dit installeerbare OpenClaw-pakket als product?" Dit verschilt van normale CI: normale CI valideert de bronstructuur, terwijl Package Acceptance één tarball valideert via hetzelfde Docker-E2E-harnas dat gebruikers na installatie of bijwerking gebruiken.

### Taken

1. `resolve_package` checkt `workflow_ref` uit, bepaalt één pakketkandidaat, schrijft `.artifacts/docker-e2e-package/openclaw-current.tgz`, schrijft `.artifacts/docker-e2e-package/package-candidate.json`, uploadt beide als het artefact `package-under-test` en vermeldt de bron, workflowreferentie, pakketreferentie, versie, SHA-256 en het profiel in de GitHub-stapsamenvatting.
2. `package_integrity` downloadt het artefact `package-under-test` en dwingt met `scripts/check-openclaw-package-tarball.mjs` het contract voor openbare pakkettarballs af.
3. `docker_acceptance` roept `openclaw-live-and-e2e-checks-reusable.yml` aan met de bepaalde bron-SHA van het pakket (met `workflow_ref` als terugval) en `package_artifact_name=package-under-test`. De herbruikbare workflow downloadt dat artefact, valideert de inventaris van de tarball, bereidt zo nodig Docker-images voor de pakketdigest voor en voert de geselecteerde Docker-lanes uit tegen dat pakket in plaats van de workflowcheckout te verpakken. Wanneer een profiel meerdere gerichte `docker_lanes` selecteert, bereidt de herbruikbare workflow het pakket en de gedeelde images eenmaal voor en spreidt die lanes vervolgens uit over parallelle, gerichte Docker-taken met unieke artefacten.
4. `package_telegram` roept optioneel `NPM Telegram Beta E2E` aan. Deze wordt uitgevoerd wanneer `telegram_mode` niet `none` is en installeert hetzelfde artefact `package-under-test` wanneer Package Acceptance er een heeft bepaald; een zelfstandige Telegram-dispatch kan nog steeds een gepubliceerde npm-specificatie installeren.
5. `summary` laat de workflow mislukken als de pakketbepaling, integriteitscontrole, Docker-acceptatie of optionele Telegram-lane is mislukt. De invoer `advisory` verlaagt acceptatiefouten tot waarschuwingen voor adviserende aanroepers.

### Kandidaatbronnen

- `source=npm` accepteert alleen `openclaw@extended-stable`, `openclaw@beta`, `openclaw@latest` of een exacte OpenClaw-releaseversie zoals `openclaw@2026.4.27-beta.2`. Gebruik dit voor gepubliceerde acceptatie van extended-stable, prerelease of stable.
- `source=ref` verpakt een vertrouwde `package_ref`-branch, -tag of volledige commit-SHA. De resolver haalt OpenClaw-branches/-tags op, controleert of de geselecteerde commit bereikbaar is vanuit de branchgeschiedenis van de repository of via een releasetag, installeert afhankelijkheden in een losgekoppelde worktree en verpakt die met `scripts/package-openclaw-for-docker.mjs`.
- `source=url` downloadt een openbare HTTPS-`.tgz`; `package_sha256` is vereist. Dit pad weigert URL-aanmeldgegevens, niet-standaard HTTPS-poorten, private/interne/voor speciaal gebruik bestemde hostnamen of omgezette IP-adressen en omleidingen die buiten hetzelfde openbare veiligheidsbeleid vallen.
- `source=trusted-url` downloadt een HTTPS-`.tgz` vanuit een benoemd beleid voor vertrouwde bronnen in `.github/package-trusted-sources.json`; `package_sha256` en `trusted_source_id` zijn vereist. Gebruik dit alleen voor door beheerders beheerde bedrijfsmirrors of private pakketrepository's waarvoor geconfigureerde hosts, poorten, padvoorvoegsels, omleidingshosts of omzetting binnen een privénetwerk nodig zijn. Als het beleid bearerauthenticatie declareert, gebruikt de workflow het vaste geheim `OPENCLAW_TRUSTED_PACKAGE_TOKEN`; in de URL ingesloten aanmeldgegevens worden nog steeds geweigerd.
- `source=artifact` downloadt één `.tgz` vanuit `artifact_run_id` en `artifact_name`; `package_sha256` is optioneel, maar moet worden opgegeven voor extern gedeelde artefacten.

Houd `workflow_ref` en `package_ref` gescheiden. `workflow_ref` is de vertrouwde workflow-/harnascode die de test uitvoert. `package_ref` is de broncommit die wordt verpakt wanneer `source=ref`. Hierdoor kan het huidige testharnas oudere vertrouwde broncommits valideren zonder oude workflowlogica uit te voeren.

### Suiteprofielen

- `smoke` — `npm-onboard-channel-agent`, `gateway-network`, `config-reload`
- `package` — `npm-onboard-channel-agent`, `doctor-switch`, `update-channel-switch`, `skill-install`, `update-corrupt-plugin`, `upgrade-survivor`, `published-upgrade-survivor`, `root-managed-vps-upgrade`, `update-restart-auth`, `plugins-offline`, `plugin-update`
- `product` — de `package`-set met live `plugins`-dekking in plaats van `plugins-offline`, plus `mcp-channels`, `cron-mcp-cleanup`, `openai-web-search-minimal`, `openwebui`
- `full` — volledige Docker-segmenten voor het releasepad met OpenWebUI
- `custom` — exact `docker_lanes`; vereist wanneer `suite_profile=custom`

Het profiel `package` gebruikt offline Plugindekking, zodat validatie van gepubliceerde pakketten niet afhankelijk is van live-beschikbaarheid van ClawHub. De optionele Telegram-lane hergebruikt het artefact `package-under-test` in `NPM Telegram Beta E2E`, waarbij het pad voor gepubliceerde npm-specificaties behouden blijft voor zelfstandige dispatches.

Zie [Updates en plugins testen](/nl/help/testing-updates-plugins) voor het speciale beleid voor het testen van updates en Plugins, inclusief lokale opdrachten,
Docker-lanes, invoer voor Package Acceptance, standaardwaarden voor releases en foutanalyse.

Releasecontroles roepen Package Acceptance aan met `source=artifact`, het voorbereide artefact met het releasepakket, `suite_profile=custom`, `docker_lanes='doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape'` en `telegram_mode=mock-openai`. Hierdoor gebruiken pakketmigratie, updates, live-installatie van Skills via ClawHub, opschoning van verouderde Plugin-afhankelijkheden, reparatie van de installatie van geconfigureerde Plugins, offline-Plugins, Plugin-updates en Telegram-bewijs allemaal dezelfde bepaalde pakkettarball. Stel na publicatie van een bèta `release_package_spec` in voor Full Release Validation of OpenClaw Release Checks om dezelfde matrix uit te voeren tegen het uitgebrachte npm-pakket zonder dit opnieuw te bouwen; stel `package_acceptance_package_spec` alleen in wanneer Package Acceptance een ander pakket nodig heeft dan de rest van de releasevalidatie. Cross-OS-releasecontroles blijven besturingssysteemspecifieke onboarding, installatieprogramma's en platformgedrag dekken; productvalidatie voor pakketten/updates moet beginnen met Package Acceptance.

De Docker-lane `published-upgrade-survivor` valideert per uitvoering één basisversie van een gepubliceerd pakket in het blokkerende releasepad. In Package Acceptance is de bepaalde tarball `package-under-test` altijd de kandidaat en selecteert `published_upgrade_survivor_baseline` de gepubliceerde basisversie waarop wordt teruggevallen, standaard `openclaw@latest`; opdrachten om mislukte lanes opnieuw uit te voeren behouden die basisversie. Full Release Validation met `run_release_soak=true` of `release_profile=full` stelt `published_upgrade_survivor_baselines='last-stable-4 2026.4.23 2026.5.2 2026.4.15'` en `published_upgrade_survivor_scenarios=reported-issues` in om uit te breiden over de vier nieuwste stabiele npm-releases, plus vastgezette grensreleases voor Plugincompatibiliteit en op problemen gebaseerde fixtures voor Feishu-configuratie, behouden bootstrap-/personabestanden, installaties van geconfigureerde OpenClaw-Plugins, logpaden met tildes en verouderde hoofdmappen met Plugin-afhankelijkheden. Overlevingsselecties voor upgrades van gepubliceerde pakketten met meerdere basisversies worden per basisversie geshard over afzonderlijke, gerichte Docker-runnertaken. De afzonderlijke workflow `Update Migration` gebruikt de Docker-lane `update-migration` met `all-since-2026.4.23`-basisversies en `plugin-deps-cleanup`-scenario's wanneer de vraag volledige opschoning van gepubliceerde updates betreft, niet de normale breedte van Full Release CI. Lokale samengevoegde uitvoeringen kunnen exacte pakketspecificaties doorgeven met `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS`, één lane behouden met `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`, zoals `openclaw@2026.4.15`, of `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` instellen voor de scenariomatrix. De gepubliceerde lane configureert de basisversie met een ingebouwd `openclaw config set`-opdrachtrecept, legt receptstappen vast in `summary.json` en test `/healthz`, `/readyz` en de RPC-status nadat de Gateway is gestart. De verse verpakte en installatieprogrammalanes voor Windows controleren ook of een geïnstalleerd pakket een overschrijving voor browserbesturing kan importeren vanaf een onbewerkt absoluut Windows-pad. De OpenAI-cross-OS-smoketest voor agentbeurten gebruikt standaard `OPENCLAW_CROSS_OS_OPENAI_MODEL` wanneer dit is ingesteld en anders `openai/gpt-5.6-luna`, zodat het bewijs voor installatie en Gateway de goedkopere GPT-5.6-testlaag gebruikt.

### Vensters voor compatibiliteit met oudere versies

Pakketacceptatie heeft begrensde compatibiliteitsvensters voor verouderde, reeds gepubliceerde pakketten. Pakketten tot en met `2026.4.25`, inclusief `2026.4.25-beta.*`, mogen het compatibiliteitspad gebruiken:

- bekende privé-QA-items in `dist/postinstall-inventory.json` mogen verwijzen naar bestanden die niet in de tarball zijn opgenomen;
- `doctor-switch` mag het persistentiesubgeval `gateway install --wrapper` overslaan wanneer het pakket die vlag niet beschikbaar stelt;
- `update-channel-switch` mag ontbrekende pnpm-`patchedDependencies` verwijderen uit de van de tarball afgeleide nep-gitfixture en mag ontbrekende persistente `update.channel` loggen;
- Plugin-smoketests mogen verouderde locaties van installatierecords lezen of ontbrekende persistentie van marketplace-installatierecords accepteren;
- `plugin-update` mag migratie van configuratiemetadata toestaan, terwijl nog steeds vereist is dat het installatierecord en het gedrag zonder herinstallatie ongewijzigd blijven.

Het gepubliceerde pakket `2026.4.26` mag ook waarschuwen voor lokale stempelbestanden met buildmetadata die al zijn uitgebracht, en pakketten tot en met `2026.5.20` mogen waarschuwen in plaats van mislukken wanneer `npm-shrinkwrap.json` ontbreekt. Latere pakketten moeten aan de moderne contracten voldoen; onder dezelfde omstandigheden treedt een fout op in plaats van een waarschuwing of het overslaan van de controle.

### Voorbeelden

```bash
# Valideer het huidige bètapakket met dekking op productniveau.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai

# Valideer het gepubliceerde extended-stable-pakket met pakketdekking.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@extended-stable \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Verpak en valideer een releasebranch met de huidige testinfrastructuur.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=ref \
  -f package_ref=release/YYYY.M.PATCH \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Valideer een tarball-URL. SHA-256 is verplicht voor source=url.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=url \
  -f package_url=https://example.com/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Valideer een tarball volgens een benoemd beleid voor vertrouwde privémirrors.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Hergebruik een tarball die door een andere Actions-run is geüpload.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=package-under-test \
  -f suite_profile=custom \
  -f docker_lanes='install-e2e plugin-update'
```

Begin bij het overzicht `resolve_package` wanneer je fouten in een mislukte pakketacceptatierun opspoort, om de pakketbron, versie en SHA-256 te bevestigen. Inspecteer daarna de onderliggende run `docker_acceptance` en de bijbehorende Docker-artefacten: `.artifacts/docker-tests/**/summary.json`, `failures.json`, lanenlogs, fasetijden en opdrachten voor opnieuw uitvoeren. Voer bij voorkeur het mislukte pakketprofiel of de exacte Docker-lanen opnieuw uit in plaats van de volledige releasevalidatie.

## Installatiesmoketest

De workflow `Install Smoke` wordt niet meer uitgevoerd voor pull requests of pushes naar `main`. Zowel de nachtelijke/handmatige wrapper als de releasevalidatie roept de alleen-lezen kern `install-smoke-reusable.yml` aan, en elke run doorloopt het volledige installatiesmoketestpad op door GitHub gehoste runners:

- De smoketestimage van het Dockerfile in de hoofdmap wordt eenmaal per doel-SHA gebouwd, gekoppeld aan de workflowrevisie en producentpoging in een onveranderlijk artefact en vervolgens geladen door de CLI-smoketest, de CLI-smoketest waarin agents de gedeelde werkruimte verwijderen, de E2E-test voor het Gateway-netwerk van de container en de smoketest voor het buildargument van de gebundelde Plugin `matrix`. De Plugin-smoketest verifieert de spiegeling van de installatie van runtimeafhankelijkheden en controleert of de Plugin wordt geladen zonder diagnostiek over het ontsnappen uit het toegangspunt.
- De installatie van het QR-pakket en de Docker-smoketests voor installatie/update (inclusief installatielanen voor Rocky Linux en een updatelaan tegen een configureerbare npm-basislijn `update_baseline_version`) worden als afzonderlijke taken uitgevoerd, zodat installatiewerk niet achter de smoketests van de hoofdimage hoeft te wachten.

De trage imageprovidersmoketest voor globale installatie met Bun wordt afzonderlijk aangestuurd door `run_bun_global_install_smoke`. Deze wordt volgens het nachtelijke schema uitgevoerd, is standaard ingeschakeld voor workflowaanroepen vanuit releasecontroles en kan worden ingeschakeld bij handmatige `Install Smoke`-dispatches. Normale PR-CI voert nog steeds de snelle regressielaan voor de Bun-launcher uit bij wijzigingen die relevant zijn voor Node. Docker-tests voor QR en installatieprogramma's behouden hun eigen, op installatie gerichte Dockerfiles.

## Lokale Docker-E2E

`pnpm test:docker:all` bouwt vooraf één gedeelde live-testimage, verpakt OpenClaw eenmaal als een npm-tarball en bouwt twee gedeelde `scripts/e2e/Dockerfile`-images:

- een kale Node/Git-runner voor installatie-, update- en Plugin-afhankelijkheidslanen;
- een functionele image die dezelfde tarball installeert in `/app` voor normale functionaliteitslanen.

Definities van Docker-lanen staan in `scripts/lib/docker-e2e-scenarios.mjs`, plannerlogica staat in `scripts/lib/docker-e2e-plan.mjs` en de runner voert alleen het geselecteerde plan uit. De planner selecteert de image per laan met `OPENCLAW_DOCKER_E2E_BARE_IMAGE` en `OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE` en voert vervolgens lanen uit met `OPENCLAW_SKIP_DOCKER_BUILD=1`.

### Instelbare waarden

| Variabele                               | Standaard | Doel                                                                                       |
| -------------------------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`      | 10      | Aantal plaatsen in de hoofdpool voor normale lanen.                                                        |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10      | Aantal plaatsen in de providergevoelige eindpool.                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`       | 9       | Limiet voor gelijktijdige live-lanen, zodat providers niet afknijpen.                                        |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`        | 5       | Limiet voor gelijktijdige npm-installatielanen.                                                              |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`    | 7       | Limiet voor gelijktijdige lanen met meerdere services.                                                            |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000    | Spreiding tussen het starten van lanen om pieken bij het aanmaken door de Docker-daemon te voorkomen; stel `0` in voor geen spreiding.     |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`  | 7200000 | Terugvaltime-out per laan (120 minuten); geselecteerde live-/eindlanen gebruiken strengere limieten.           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`          | niet ingesteld   | `1` drukt het plannerplan af zonder lanen uit te voeren.                                          |
| `OPENCLAW_DOCKER_ALL_LANES`            | niet ingesteld   | Door komma's gescheiden lijst met exacte lanen; slaat de opschoonsmoketest over, zodat agents één mislukte laan kunnen reproduceren. |

Een laan die zwaarder is dan de effectieve limiet kan nog steeds vanuit een lege pool starten en wordt daarna alleen uitgevoerd totdat capaciteit wordt vrijgegeven. De lokale aggregatierun controleert Docker vooraf, verwijdert verouderde OpenClaw-E2E-containers, toont de status van actieve lanen, bewaart laantijden voor ordening van langste naar kortste en stopt standaard na de eerste fout met het plannen van nieuwe gepoolde lanen.

### Herbruikbare live-/E2E-workflow

De herbruikbare live-/E2E-workflow vraagt `scripts/test-docker-all.mjs --plan-json` welk pakket, imagetype, welke live-image, laan en dekking van inloggegevens vereist zijn. `scripts/docker-e2e.mjs` zet dat plan vervolgens om in GitHub-uitvoerwaarden en overzichten. De workflow verpakt OpenClaw via `scripts/package-openclaw-for-docker.mjs`, downloadt een pakketartefact uit de huidige run of downloadt een pakketartefact uit `package_artifact_run_id` en valideert vervolgens de inhoud van de tarball. Het standaardpad `no-push-artifact` bouwt kale/functionele images met tags op basis van de pakketdigest via Blacksmiths cache voor Docker-lagen, verpakt de exacte imagebytes in een onveranderlijk workflowartefact en laat elke consument dat artefact verifiëren en laden. `existing-only` vereist daarentegen expliciete GHCR-verwijzingen `docker_e2e_bare_image`/`docker_e2e_functional_image` en bouwt of pusht nooit. Deze registerdownloads gebruiken een begrensde time-out van 180 seconden per poging, zodat een vastgelopen stream snel opnieuw wordt geprobeerd in plaats van het grootste deel van het kritieke pad van de CI-pijplijn in beslag te nemen. Na een geslaagde geplande validatie geeft `openclaw-scheduled-live-checks.yml` het onveranderlijke manifest van de geteste images door aan de afzonderlijke publicatiefunctie met schrijfrechten voor pakketten; alleen-lezen aanroepers voor releases en prereleases doorlopen die schrijver nooit.

### Segmenten van het releasepad

De Docker-dekking van releases voert kleinere gesegmenteerde taken uit met `OPENCLAW_SKIP_DOCKER_BUILD=1`, zodat elk segment alleen het door artefacten ondersteunde imagetype verifieert en laadt dat het nodig heeft (of het downloadt bij expliciet hergebruik via `existing-only`) en meerdere lanen uitvoert via dezelfde gewogen planner:

- `OPENCLAW_DOCKER_ALL_PROFILE=release-path`
- `OPENCLAW_DOCKER_ALL_CHUNK=core | package-update-openai | package-update-anthropic | package-update-core | plugins-runtime-plugins | plugins-runtime-services | plugins-runtime-install-a..h | openwebui`

De huidige Docker-segmenten voor releases zijn `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, `plugins-runtime-install-a` tot en met `plugins-runtime-install-h`, en `openwebui`. `package-update-openai` bevat de live-pakketlaan voor de Codex-Plugin, die het kandidaatpakket van OpenClaw installeert, de Codex-Plugin installeert vanuit `codex_plugin_spec` of een tarball van dezelfde verwijzing met expliciete goedkeuring voor installatie van de Codex-CLI, voorafgaande controles van de Codex-CLI en agentbeurten binnen dezelfde sessie uitvoert en vervolgens een beurt met gemiddeld denkniveau zonder nieuwe poging uitvoert die voortgang verzendt, willekeurige invoer uit de werkruimte leest, het exacte artefact daarvan schrijft en de voltooiing verzendt. `plugins-runtime-core`, `plugins-runtime` en `plugins-integrations` blijven geaggregeerde aliassen voor Plugins/runtime. De laanalias `install-e2e` blijft de geaggregeerde alias voor het handmatig opnieuw uitvoeren van beide providerinstallatielanen.

OpenWebUI wordt uitgevoerd als een zelfstandig segment `openwebui` op een speciale Blacksmith-runner met een grote schijf wanneer dekking voor een stabiele of volledige release dit vereist, zelfs wanneer de herbruikbare workflow ondersteunde taken naar door GitHub gehoste runners routeert. Door de download van de externe image gescheiden te houden, concurreert de grote image niet met de gedeelde pakket- en Plugin-images in `plugins-runtime-services`; verouderde geaggregeerde Plugin-/runtimesegmenten bevatten OpenWebUI nog steeds voor compatibele handmatige heruitvoeringen. Updatelanen voor gebundelde kanalen proberen bij tijdelijke npm-netwerkfouten eenmaal opnieuw.

Elk segment uploadt `.artifacts/docker-tests/` met laanlogs, tijden, `summary.json`, `failures.json`, fasetijden, JSON van het plannerplan, tabellen met trage lanen en opdrachten om elke laan opnieuw uit te voeren. De workflowinvoer `docker_lanes` voert geselecteerde lanen uit tegen images die voor die run zijn voorbereid in plaats van tegen de segmenttaken, waardoor het opsporen van fouten in mislukte lanen beperkt blijft tot één gerichte Docker-taak; als een geselecteerde laan een live-Docker-laan is, bouwt de gerichte taak voor die heruitvoering lokaal de live-testimage. De helper voor opnieuw uitvoeren valideert de exact geselecteerde doel-SHA van het foutartefact en een handmatige dispatch verpakt die verwijzing opnieuw, omdat de interne pakketcombinatie van de herbruikbare workflow geen deel uitmaakt van het schema `workflow_dispatch`. Gegenereerde opdrachten bevatten voorbereide image-invoerwaarden en alleen `shared_image_policy=existing-only` wanneer die invoerwaarden door GHCR worden ondersteund; artefacttags die lokaal bij de runner horen, worden weggelaten zodat een nieuwe runner ze opnieuw bouwt. Een expliciete doeloverschrijving verwijdert herstelde GHCR-imageverwijzingen, tenzij het artefact bewijst dat ze met de overschrijving overeenkomen. Ook door artefacten gegenereerde verwijzingen naar workflowdefinities worden weggelaten omdat tijdelijke branches voor volledige releases worden verwijderd; dispatch gebruikt de standaardbranch van de repository, tenzij de operator deze expliciet overschrijft.

```bash
pnpm test:docker:rerun <run-id>      # download Docker-artefacten en druk gecombineerde/gerichte opdrachten per laan voor opnieuw uitvoeren af
pnpm test:docker:timings <summary>   # overzichten van trage lanen en het kritieke pad per fase
```

De geplande live-/E2E-workflow voert dagelijks de volledige Docker-testsuite van het releasepad uit en roept na een geslaagde uitvoering de expliciete publicatiefunctie aan voor de exact geteste imageartefacten.

## Plugin-prerelease

`Plugin Prerelease` biedt duurdere product-/pakketdekking en is daarom een afzonderlijke workflow die wordt gestart door `Full Release Validation` of door een expliciete operator. Bij normale pull requests, pushes naar `main` en zelfstandige handmatige CI-starts blijft die suite uitgeschakeld. De tests voor gebundelde plugins worden verdeeld over acht extensieworkers; deze extensieshardtaken voeren maximaal twee pluginconfiguratiegroepen tegelijk uit, met één Vitest-worker per groep en een grotere Node-heap, zodat pluginbatches met veel imports geen extra CI-taken veroorzaken. Het prereleasepad voor Docker dat alleen voor releases wordt gebruikt (ingeschakeld via de invoer `full_release_validation`) bundelt gerichte Docker-lanes in groepen van vier, om te voorkomen dat tientallen runners worden gereserveerd voor taken van één tot drie minuten. De workflow uploadt ook een informatief `plugin-inspector-advisory`-artefact vanuit `@openclaw/plugin-inspector`; bevindingen van de inspecteur dienen als invoer voor triage en veranderen niets aan de blokkerende Plugin Prerelease-gate.

## QA Lab

QA Lab heeft speciale CI-lanes buiten de primaire slim afgebakende workflow. Agentische pariteit valt onder de brede QA- en releaseharnassen en is geen zelfstandige PR-workflow. Gebruik `Full Release Validation` met `rerun_group=qa-parity` wanneer pariteit deel moet uitmaken van een brede validatierun.

- De workflow `QA-Lab - All Lanes` wordt elke nacht uitgevoerd op `main` en bij handmatige start; deze splitst uit naar mockpariteit plus live-taken voor Matrix, Telegram, Discord, WhatsApp en Slack. Live-taken gebruiken de omgeving `qa-live-shared`; Telegram, Discord, WhatsApp en Slack gebruiken Convex-leases, terwijl Matrix tijdelijke lokale aanmeldgegevens inricht.

Releasecontroles voeren live-transportlanes voor Matrix en Telegram uit met de deterministische mockprovider en als mock gekwalificeerde modellen (`mock-openai/gpt-5.6-luna` en `mock-openai/gpt-5.6-luna-alt`), zodat het kanaalcontract is geïsoleerd van de latentie van live-modellen en het normale opstarten van providerplugins. De live-transportgateway schakelt geheugenzoekopdrachten uit, omdat QA-pariteit geheugengedrag afzonderlijk dekt; providerconnectiviteit wordt gedekt door de afzonderlijke suites voor live-modellen, native providers en Docker-providers.

Geplande Matrix-gates en Matrix-gates voor releases gebruiken de gedeelde QA Lab-suitehost en live-adapter met de releasescenario's. De standaardwaarde van de CLI en de handmatige workflowinvoer blijven `all`; handmatige starts van `all` splitsen uit naar de profielen `transport`, `media`, `e2ee-smoke`, `e2ee-deep` en `e2ee-cli`, zodat het bewijs met 93 scenario's binnen de time-outs per taak blijft. Gerichte handmatige starts selecteren `fast`, `release` of `transport` in één taak.

`OpenClaw Release Checks` voert vóór releasegoedkeuring ook de releasekritieke QA Lab-lanes uit; de gate voor QA-pariteit voert de kandidaat- en baselinepakketten uit als parallelle lanetaken en downloadt vervolgens beide artefacten naar een kleine rapporttaak voor de uiteindelijke pariteitsvergelijking.

Volg voor normale PR's afgebakend CI-/controlebewijs in plaats van pariteit als een vereiste status te behandelen.

## CodeQL

De workflow `CodeQL` is bewust een beperkte beveiligingsscanner voor een eerste controle en geen volledige scan van de repository. Dagelijkse, handmatige, push- en niet-concept-pull-requestbewakingsruns voor `main` scannen Actions-workflowcode plus de JavaScript-/TypeScript-oppervlakken met het hoogste risico, met uiterst betrouwbare beveiligingsquery's die zijn gefilterd op hoge/kritieke `security-severity`.

De bewaking voor pull requests blijft licht: deze start alleen voor wijzigingen onder `.github/actions`, `.github/codeql`, `.github/workflows`, `packages`, `scripts`, `src` of runtimepaden van procesbeherende gebundelde plugins, en voert dezelfde uiterst betrouwbare beveiligingsmatrix uit als de geplande workflow. CodeQL voor Android en macOS blijft buiten de standaardinstellingen voor PR's.

### Beveiligingscategorieën

| Categorie                                          | Oppervlak                                                                                                                             |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-security-high/core-auth-secrets`         | Authenticatie, geheimen, sandbox, cron en Gateway-baseline                                                                                  |
| `/codeql-security-high/channel-runtime-boundary`  | Implementatiecontracten voor kernkanalen plus de runtime van kanaalplugins, Gateway, Plugin SDK, geheimen en auditcontactpunten              |
| `/codeql-security-high/network-ssrf-boundary`     | Oppervlakken voor SSRF in de kern, IP-parsering, netwerkbewaking, ophalen via het web en SSRF-beleid van de Plugin SDK                                                |
| `/codeql-security-high/mcp-process-tool-boundary` | MCP-servers, helpers voor procesuitvoering, uitgaande levering en gates voor de uitvoering van agenttools                                           |
| `/codeql-security-high/process-exec-boundary`     | Lokale shell, helpers voor het starten van processen, runtimes van procesbeherende gebundelde plugins en koppellogica voor workflowscripts                             |
| `/codeql-security-high/plugin-trust-boundary`     | Vertrouwensoppervlakken voor plugininstallatie, loader, manifest, register, installatie via pakketbeheer, bronladen en pakketcontracten van de Plugin SDK |

### Platformspecifieke beveiligingsshards

- `CodeQL Android Critical Security` — geplande Android-beveiligingsshard. Bouwt de Android-app handmatig voor CodeQL op de kleinste Blacksmith Linux-runner die door de workflowcontrole wordt geaccepteerd. Uploadt onder `/codeql-critical-security/android`.
- `CodeQL macOS Critical Security` — wekelijkse/handmatige macOS-beveiligingsshard. Bouwt de macOS-app handmatig voor CodeQL op Blacksmith macOS, filtert bouwresultaten van afhankelijkheden uit de geüploade SARIF en uploadt onder `/codeql-critical-security/macos`. Blijft buiten de dagelijkse standaardinstellingen omdat de macOS-build de uitvoeringstijd domineert, zelfs als er geen bevindingen zijn.

### Categorieën voor kritieke kwaliteit

`CodeQL Critical Quality` is de overeenkomstige niet-beveiligingsshard. Deze voert alleen JavaScript-/TypeScript-kwaliteitsquery's zonder beveiligingsfocus en met fout-ernst uit op beperkte hoogwaardige oppervlakken op door GitHub gehoste Linux-runners, zodat kwaliteitsscans geen Blacksmith-budget voor runnerregistratie verbruiken. De bewaking voor pull requests is bewust kleiner dan het geplande profiel: niet-concept-PR's voeren alleen de overeenkomstige shards uit voor de oppervlakken die ze raken, uit dertien naar PR's routeerbare shards — `agent-runtime-boundary`, `channel-runtime-boundary`, `config-boundary`, `core-auth-secrets`, `gateway-runtime-boundary`, `mcp-process-runtime-boundary`, `memory-runtime-boundary`, `network-runtime-boundary`, `plugin-boundary`, `plugin-sdk-package-contract`, `plugin-sdk-reply-runtime`, `provider-runtime-boundary` en `session-diagnostics-boundary`. `ui-control-plane` en `web-media-runtime-boundary` blijven buiten PR-runs. Wijzigingen in de CodeQL-configuratie en kwaliteitsworkflow voeren de volledige PR-shardset uit (de shard voor de netwerkruntime wordt geactiveerd door zijn eigen CodeQL-configuratiebestanden en bronpaden die eigenaar zijn van netwerkfunctionaliteit).

Handmatig starten accepteert:

```text
profile=all|agent-runtime-boundary|config-boundary|core-auth-secrets|channel-runtime-boundary|gateway-runtime-boundary|memory-runtime-boundary|mcp-process-runtime-boundary|network-runtime-boundary|plugin-boundary|plugin-sdk-package-contract|plugin-sdk-reply-runtime|provider-runtime-boundary|session-diagnostics-boundary
```

De beperkte profielen zijn leer-/iteratiehooks om één kwaliteitsshard geïsoleerd uit te voeren.

| Categorie                                                | Oppervlak                                                                                                                                                           |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-critical-quality/core-auth-secrets`            | Code voor de beveiligingsgrens van authenticatie, geheimen, sandbox, cron en Gateway                                                                                                  |
| `/codeql-critical-quality/config-boundary`              | Contracten voor configuratieschema's, migratie, normalisatie en IO                                                                                                         |
| `/codeql-critical-quality/gateway-runtime-boundary`     | Gateway-protocolschema's en servermethodecontracten                                                                                                              |
| `/codeql-critical-quality/channel-runtime-boundary`     | Implementatiecontracten voor kernkanalen en gebundelde kanaalplugins                                                                                                  |
| `/codeql-critical-quality/agent-runtime-boundary`       | Contracten voor opdrachtuitvoering, model-/providerdispatch, automatische antwoorddispatch en wachtrijen, en de runtime van het ACP-besturingsvlak                                               |
| `/codeql-critical-quality/mcp-process-runtime-boundary` | MCP-servers en toolbridges, helpers voor procestoezicht en contracten voor uitgaande levering                                                                        |
| `/codeql-critical-quality/memory-runtime-boundary`      | SDK voor de geheugenhost, geheugenruntimefacades, geheugenaliassen van de Plugin SDK, activeringslogica van de geheugenruntime en doctor-opdrachten voor geheugen                                    |
| `/codeql-critical-quality/network-runtime-boundary`     | Netwerkbeleidspakket, runtime voor onbewerkte sockets en proxyvastlegging, SSH-tunnel, Gateway-vergrendeling, JSONL-socket en oppervlakken voor pushtransport                                 |
| `/codeql-critical-quality/session-diagnostics-boundary` | Interne antwoordwachtrijen, sessieleveringswachtrijen, helpers voor binding/levering van uitgaande sessies, oppervlakken voor diagnostische gebeurtenissen/logbundels en CLI-contracten voor sessiedoctor |
| `/codeql-critical-quality/plugin-sdk-reply-runtime`     | Inkomende antwoorddispatch van de Plugin SDK, helpers voor antwoordpayloads/-segmentering/runtime, opties voor kanaalantwoorden, leveringswachtrijen en helpers voor sessie-/threadbinding             |
| `/codeql-critical-quality/provider-runtime-boundary`    | Normalisatie van modelcatalogi, providerauthenticatie en -detectie, registratie van providerruntimes, standaardwaarden/catalogi van providers en registers voor web/zoeken/ophalen/embeddings    |
| `/codeql-critical-quality/ui-control-plane`             | Opstarten van de Control UI, lokale persistentie, Gateway-besturingsstromen en runtimecontracten van het taakbesturingsvlak                                                          |
| `/codeql-critical-quality/web-media-runtime-boundary`   | Runtimecontracten voor ophalen/zoeken via het web in de kern, media-IO, mediabegrip, beeldgeneratie en mediageneratie                                                    |
| `/codeql-critical-quality/plugin-boundary`              | Contracten voor loader, register, openbaar oppervlak en toegangspunten van de Plugin SDK                                                                                             |
| `/codeql-critical-quality/plugin-sdk-package-contract`  | Gepubliceerde broncode van de Plugin SDK aan pakketzijde en helpers voor pluginpakketcontracten                                                                                      |

Kwaliteit blijft gescheiden van beveiliging, zodat kwaliteitsbevindingen kunnen worden gepland, gemeten, uitgeschakeld of uitgebreid zonder het beveiligingssignaal te vertroebelen. Uitbreiding van CodeQL voor Swift, Python en gebundelde plugins mag pas weer als afgebakend of geshard vervolgwerk worden toegevoegd nadat de beperkte profielen een stabiele uitvoeringstijd en een stabiel signaal hebben.

## Onderhoudsworkflows

### Docs Agent

De workflow `Docs Agent` is een gebeurtenisgestuurde Codex-onderhoudslane om bestaande documentatie afgestemd te houden op recent doorgevoerde wijzigingen. Er is geen zuiver schema: een geslaagde CI-run voor een push door een niet-bot op `main` kan deze activeren, en met een handmatige start kan deze rechtstreeks worden uitgevoerd. Aanroepen vanuit workflowruns worden overgeslagen wanneer `main` verder is gegaan of wanneer in het afgelopen uur een andere niet-overgeslagen Docs Agent-run is aangemaakt. Wanneer de workflow wordt uitgevoerd, beoordeelt deze het commitbereik van de vorige bron-SHA van een niet-overgeslagen Docs Agent-run tot de huidige `main`, zodat één uurlijkse run alle wijzigingen aan main kan dekken die sinds de vorige documentatiecontrole zijn verzameld.

### Testprestatie-agent

De `Test Performance Agent`-workflow is een gebeurtenisgestuurde Codex-onderhoudslane voor trage tests. Deze heeft geen zuiver tijdschema: een geslaagde niet-bot-push-CI-run op `main` kan de workflow activeren, maar deze wordt overgeslagen als die UTC-dag al een andere workflow-run-aanroep is uitgevoerd of actief is. Handmatige activering omzeilt die dagelijkse activiteitscontrole. De lane stelt een gegroepeerd Vitest-prestatierapport voor de volledige suite samen, laat Codex alleen kleine testprestatieverbeteringen aanbrengen die de dekking behouden in plaats van brede refactors, voert vervolgens het rapport voor de volledige suite opnieuw uit en weigert wijzigingen die het basisaantal geslaagde tests verlagen. Het gegroepeerde rapport registreert per configuratie de verstreken tijd en maximale RSS op Linux en macOS, zodat de vergelijking voor en na de wijzigingen verschillen in testgeheugengebruik naast verschillen in duur toont. Als de basislijn falende tests bevat, mag Codex alleen duidelijke fouten herstellen en moet het rapport voor de volledige suite na de agent slagen voordat iets wordt gecommit. Wanneer `main` verdergaat voordat de bot-push is geland, rebaset de lane de gevalideerde patch, voert `pnpm check:changed` opnieuw uit en probeert de push opnieuw; conflicterende verouderde patches worden overgeslagen. De lane gebruikt door GitHub gehoste Ubuntu, zodat de Codex-action dezelfde drop-sudo-veiligheidsinstelling kan behouden als de documentatieagent.

### Dubbele PR's na samenvoegen

De `Duplicate PRs After Merge`-workflow is een handmatige onderhoudersworkflow voor het opschonen van duplicaten na het landen. Deze gebruikt standaard een proefuitvoering en sluit alleen expliciet vermelde PR's wanneer `apply=true`. Voordat GitHub wordt gewijzigd, controleert de workflow of de gelande PR is samengevoegd en of elk duplicaat een gedeeld gerefereerd issue of overlappende gewijzigde hunks heeft.

```bash
gh workflow run duplicate-after-merge.yml \
  -f landed_pr=70532 \
  -f duplicate_prs='70530,70592' \
  -f apply=true
```

## Lokale controlegates en routering van wijzigingen

### Ratchet voor het aantal configuratiebasislijnen

`pnpm config:docs:check` weigert ongedocumenteerde groei van het configuratieoppervlak en beschadigde of verouderde momentopnamen van aantallen. Wanneer een beoordeelde productwijziging opzettelijk schemapaden toevoegt, voer je `pnpm config:docs:gen` uit, inspecteer je de verschillen in aantallen voor core/channel/plugin en de gegenereerde SHA-256-bestanden, en commit je de bewuste verhoging van de basislijn samen met het schema, de hulptekst, labels, migratie en tests. Bewerk het bestand met aantallen niet handmatig om de ratchet te omzeilen.

Configuratieauteurs moeten nieuwe bladeren ook in niveaus indelen voor Settings. Voeg `advanced: false` of
`advanced: true` toe aan het blad, of plaats de sleutel onder een voorouder waarvan alle
afstammelingen het niveau moeten overnemen. Niet-geclassificeerde wortels laten de
schemakwaliteitstest mislukken met kopieer-en-plak-stubs; paden zonder voorouder
worden standaard als geavanceerd ingedeeld. De gecureerde momentopname van veelvoorkomende
bladeren maakt opzettelijke niveauwijzigingen zichtbaar tijdens de review.

De lokale logica voor gewijzigde lanes bevindt zich in `scripts/changed-lanes.mjs` en wordt uitgevoerd door `scripts/check-changed.mjs`. Die lokale controlegate is strenger ten aanzien van architectuurgrenzen dan het brede platformbereik van CI:

- productiewijzigingen in core voeren typecontroles voor core-productie en core-tests uit, plus core-lint/guards;
- wijzigingen die uitsluitend core-tests betreffen, voeren alleen de typecontrole voor core-tests plus core-lint uit;
- productiewijzigingen in extensies voeren typecontroles voor extensieproductie en extensietests uit, plus extensielint;
- wijzigingen die uitsluitend extensietests betreffen, voeren de typecontrole voor extensietests plus extensielint uit;
- wijzigingen aan de openbare Plugin-SDK of plugincontracten breiden uit naar typecontrole van extensies, omdat extensies afhankelijk zijn van die core-contracten (Vitest-sweeps van extensies blijven expliciet testwerk);
- versieverhogingen die uitsluitend releasemetadata betreffen, voeren gerichte controles van versie/configuratie/rootafhankelijkheden uit;
- onbekende wijzigingen aan de root/configuratie schakelen voor de veiligheid alle controlelanes in.

De lokale routering van gewijzigde tests bevindt zich in `scripts/test-projects.test-support.mjs` en is opzettelijk goedkoper dan `check:changed`: rechtstreeks gewijzigde tests voeren zichzelf uit, bronwijzigingen geven de voorkeur aan expliciete toewijzingen en daarna aan naastgelegen tests en afhankelijken in de importgrafiek. Gedeelde configuratie voor levering in groepsruimten is een van de expliciete toewijzingen: wijzigingen aan de configuratie voor zichtbare antwoorden in groepen, de leveringsmodus voor bronantwoorden of de systeemprompt van de berichttool worden gerouteerd via de core-antwoordtests plus leveringsregressies voor Discord en Slack, zodat een wijziging aan een gedeelde standaardwaarde al vóór de eerste PR-push faalt. Gebruik `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` alleen wanneer de wijziging zo breed voor de testinfrastructuur geldt dat de goedkope toegewezen set geen betrouwbare benadering is.

## Testbox-validatie

Crabbox is de door de repository beheerde wrapper voor externe machines voor Linux-bewijs door onderhouders. Agentsessies
houden één of enkele gerichte tests en goedkope statische controles alleen lokaal voor
vertrouwde broncode wanneer de bestaande installatie van afhankelijkheden gereed is. Ze gebruiken Crabbox voor grotere suites en
rekenintensief werk, waaronder builds, typecontroles, lint-fan-out,
Docker, pakketlanes, E2E, live-bewijs en CI-pariteit. Zwaar bewijs door vertrouwde onderhouders
gebruikt standaard `blacksmith-testbox`, en `.crabbox.yaml` gebruikt dit nu ook standaard. De geconfigureerde
workflow voorziet in provider- en agentreferenties, dus niet-vertrouwde code van bijdragers of
forks moet in plaats daarvan CI zonder geheimen voor forks of opgeschoonde rechtstreekse AWS-Crabbox gebruiken.
Opgeschoonde AWS-runs stellen `CRABBOX_ENV_ALLOW=CI` in, geven
`--no-hydrate` door en gebruiken een nieuwe tijdelijke externe `HOME`; dit voorkomt dat de
`OPENCLAW_*`-toestaanlijst van de repository en bestaande authenticatieprofielen niet-vertrouwde code bereiken.
Ze gebruiken een nieuw opgewarmde lease die specifiek voor die niet-vertrouwde broncode is bedoeld, nooit een
vertrouwde of eerder van referenties voorziene lease. Start een geïnstalleerd vertrouwd Crabbox-
binair bestand vanuit een schone vertrouwde `main`-checkout en haal alleen de externe PR op met
`--fresh-pr`; voer de wrapper of configuratie van de niet-vertrouwde checkout nooit lokaal uit.
Maak `CRABBOX_AWS_INSTANCE_PROFILE` ongedaan en stop veilig tenzij de opgeloste
`aws.instanceProfile` leeg is. Gebruik vóór elke installatie/test vertrouwde
tools met absolute paden om een IMDSv2-token te vereisen, aan te tonen dat het eindpunt voor IAM-referenties
404 retourneert en externe `git rev-parse HEAD` te vergelijken met de volledige
beoordeelde SHA van de PR-head. Koppel de lease aan die SHA en stop/warm opnieuw op bij een wijziging van de head.
Upload vertrouwde `scripts/crabbox-untrusted-bootstrap.sh` vanuit schone `main`
naast `--fresh-pr`; dit installeert vastgezette Node/pnpm-versies, verifieert de SHA en
de vastgezette pakketbeheerder, isoleert `HOME`, installeert afhankelijkheden en voert vervolgens de
gevraagde test uit.
Maak alle `CRABBOX_TAILSCALE*`-overschrijvingen ongedaan, dwing `--network public
--tailscale=false` af, wis exit-node-/LAN-vlaggen en vereis dat `crabbox inspect`
openbare netwerktoegang zonder Tailscale-status rapporteert voordat een script wordt geüpload.
Eigen AWS-/Hetzner-capaciteit blijft ook de uitwijkmogelijkheid voor Blacksmith-storingen,
quotaproblemen of expliciete tests met eigen capaciteit.

Agents warmen niet vooraf op voor verwacht werk. Verkrijg pas een Testbox wanneer de
eerste zware opdracht gereed is, hergebruik de geretourneerde `tbx_...`-id voor latere zware
opdrachten, synchroniseer bij elke run de huidige checkout en stop deze vóór de overdracht.

Door Crabbox ondersteunde Blacksmith-runs warmen Testboxes voor eenmalig gebruik op, claimen en synchroniseren ze, voeren opdrachten uit, rapporteren en ruimen op.
De ingebouwde synchronisatiecontrole mislukt snel wanneer
`git status --short` op de gesynchroniseerde box ten minste 200 verwijderde bijgehouden bestanden toont,
waardoor verdwijnende rootbestanden zoals `pnpm-lock.yaml` worden gedetecteerd. Stel voor opzettelijke
PR's met veel verwijderingen `CRABBOX_ALLOW_MASS_DELETIONS=1` in voor de externe opdracht.

Crabbox beëindigt ook een lokale aanroep van de Blacksmith-CLI die langer dan vijf minuten
in de synchronisatiefase blijft zonder uitvoer na de synchronisatie. Stel
`CRABBOX_BLACKSMITH_SYNC_TIMEOUT_MS=0` in om die bewaking uit te schakelen, of gebruik een grotere
waarde in milliseconden voor ongewoon grote lokale diffs.

Controleer vóór een eerste run de wrapper vanuit de repositoryroot:

```bash
pnpm crabbox:run -- --help | sed -n '1,120p'
```

De repositorywrapper weigert een verouderd Crabbox-binair bestand dat de geselecteerde provider niet vermeldt, en door Blacksmith ondersteunde runs vereisen Crabbox 0.22.0 of nieuwer, zodat de wrapper het huidige gedrag voor Testbox-synchronisatie, wachtrijen en opschoning krijgt. Vermijd in Codex-worktrees of gekoppelde/sparse checkouts het lokale `pnpm crabbox:run`-script, omdat pnpm afhankelijkheden kan reconciliëren voordat Crabbox start; roep in plaats daarvan de node-wrapper rechtstreeks aan:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --timing-json --shell -- "pnpm test <path-or-filter>"
```

Bouw bij gebruik van de naastgelegen checkout het genegeerde lokale binaire bestand opnieuw vóór timing- of bewijswerk:

```bash
version="$(git -C ../crabbox describe --tags --always --dirty | sed 's/^v//')" \
  && go build -C ../crabbox -trimpath -ldflags "-s -w -X github.com/openclaw/crabbox/internal/cli.version=${version}" -o bin/crabbox ./cmd/crabbox
```

Het `blacksmith:`-blok in `.crabbox.yaml` zet de standaardwaarden voor organisatie, workflow, taak en ref al vast, dus de expliciete vlaggen hieronder zijn optioneel. Gate voor wijzigingen:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --blacksmith-org openclaw \
  --blacksmith-workflow .github/workflows/ci-check-testbox.yml \
  --blacksmith-job check \
  --blacksmith-ref main \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm check:changed"
```

Gerichte heruitvoering van tests op Testbox wanneer lokale afhankelijkheden niet beschikbaar zijn of het
doel uitwaaiert:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test <path-or-filter>"
```

Volledige suite:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test"
```

Lees de uiteindelijke JSON-samenvatting. De nuttige velden zijn `provider`, `leaseId`,
`syncDelegated`, `exitCode`, `commandMs` en `totalMs`. Voor gedelegeerde
Blacksmith-Testbox-runs vormen de afsluitcode en JSON-samenvatting van de Crabbox-wrapper het
opdrachtresultaat. De gekoppelde GitHub Actions-run beheert de voorziening en keepalive; deze
kan eindigen als `cancelled` wanneer de Testbox extern wordt gestopt nadat de SSH-
opdracht al is teruggekeerd. Behandel dit als een opschonings-/statusartefact, tenzij
de wrapperwaarde `exitCode` niet nul is of de opdrachtuitvoer een mislukte test toont.
Eenmalige door Blacksmith ondersteunde Crabbox-runs horen de Testbox automatisch te stoppen;
als een run wordt onderbroken of de opschoning onduidelijk is, inspecteer je actieve boxen en stop je alleen
de boxen die je zelf hebt aangemaakt:

```bash
blacksmith testbox list --all
blacksmith testbox status --id <tbx_id>
blacksmith testbox stop --id <tbx_id>
```

Gebruik hergebruik alleen wanneer je bewust meerdere opdrachten op dezelfde van referenties voorziene box nodig hebt:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --id <tbx_id> --timing-json --shell -- "corepack pnpm test <path-or-filter>"
pnpm crabbox:stop -- <tbx_id>
```

Hergebruik de lease, niet verouderde broncode. Laat `--no-sync` weg, zodat elke run de
huidige checkout uploadt; gebruik dit alleen om bewust een ongewijzigde, al gesynchroniseerde tree
opnieuw uit te voeren. Niet-vertrouwde code van bijdragers/forks moet voor elke opdracht
`CRABBOX_ENV_ALLOW=CI`, `--provider aws --no-hydrate` en een nieuwe
tijdelijke externe `HOME` gebruiken; installeer afhankelijkheden binnen die
opgeschoonde opdracht voordat je test. Hergebruik alleen een nieuw opgewarmde lease die specifiek is bedoeld voor
dezelfde niet-vertrouwde broncode; nooit een vertrouwde of eerder van referenties voorziene lease. Voer
de wrapper of configuratie van de niet-vertrouwde checkout nooit lokaal uit: start het geïnstalleerde
vertrouwde Crabbox-binaire bestand vanuit schone vertrouwde `main` en geef bij elke
run `--fresh-pr` door. Houd `CRABBOX_AWS_INSTANCE_PROFILE` oningesteld, weiger een niet-leeg opgelost
instantieprofiel, vereis een vertrouwd extern IMDS-bewijs zonder rol en verifieer de
beoordeelde head-SHA vóór installatie/test. Koppel de lease aan die SHA; stop en
warm opnieuw op na elke wijziging van de head. Gebruik CI zonder geheimen voor forks als er geen externe PR bestaat.
Selecteer nooit `hydrate-github` of de van referenties voorziene Blacksmith-workflow
voor niet-vertrouwde broncode.

Als Crabbox de defecte laag is maar Blacksmith zelf werkt, gebruik je rechtstreeks
Blacksmith alleen voor diagnostiek zoals `list`, `status` en opschoning. Herstel het
Crabbox-pad voordat je een rechtstreekse Blacksmith-run als onderhoudersbewijs beschouwt.

Als `blacksmith testbox list --all` en `blacksmith testbox status` werken, maar nieuwe
warm-ups na een paar minuten nog steeds `queued` zijn zonder IP-adres of URL van een Actions-run,
beschouw dit dan als druk op de Blacksmith-provider, wachtrij, facturering of organisatielimiet. Stop de
in de wachtrij geplaatste id's die je hebt aangemaakt, start geen nieuwe Testboxes en verplaats het bewijs naar het
onderstaande pad voor eigen Crabbox-capaciteit, terwijl iemand het Blacksmith-dashboard,
de facturering en organisatielimieten controleert.

Schaal alleen op naar eigen Crabbox-capaciteit wanneer Blacksmith niet beschikbaar is, door quota wordt beperkt, de benodigde omgeving mist of eigen capaciteit expliciet het doel is:

```bash
CRABBOX_CAPACITY_REGIONS=eu-west-1,eu-west-2,eu-central-1,us-east-1,us-west-2 \
  pnpm crabbox:warmup -- --provider aws --class standard --market on-demand --idle-timeout 90m
pnpm crabbox:hydrate -- --provider aws --id <cbx_id-or-slug>
pnpm crabbox:run -- --provider aws --id <cbx_id-or-slug> --timing-json --shell -- "pnpm check:changed"
pnpm crabbox:stop -- --provider aws <cbx_id-or-slug>
```

Vermijd bij druk op AWS `class=beast`, tenzij de taak echt CPU-capaciteit van de 48xlarge-klasse vereist. Een `beast`-aanvraag begint bij 192 vCPU's en is de eenvoudigste manier om het regionale EC2 Spot- of On-Demand Standard-quotum te overschrijden. De door de repository beheerde `.crabbox.yaml` gebruikt standaard `class: standard`, de on-demand-markt en `capacity.hints: true`, zodat bemiddelde AWS-leases de geselecteerde regio/markt, quotumdruk, Spot-terugval en waarschuwingen voor klassen met hoge druk weergeven. Gebruik `fast` voor zwaardere, brede controles, `large` alleen wanneer standard/fast niet volstaan en `beast` alleen voor uitzonderlijke CPU-gebonden lanes, zoals volledige testsuites of Docker-matrices voor alle plugins, expliciete release-/blokkeringsvalidatie of prestatieprofilering met veel cores. Gebruik `beast` niet voor `pnpm check:changed`, gerichte tests, werk dat alleen documentatie betreft, gewone lint-/typecontroles, kleine E2E-reproducties of triage van een Blacksmith-storing. Gebruik `--market on-demand` voor capaciteitsdiagnose, zodat schommelingen in de Spot-markt niet met het signaal worden vermengd.

`.crabbox.yaml` beheert de standaardinstellingen voor provider, synchronisatie en hydratatie van GitHub Actions. Crabbox-synchronisatie draagt `.git` nooit over, zodat de gehydrateerde Actions-checkout zijn eigen externe Git-metagegevens behoudt in plaats van lokale maintainer-remotes en objectstores te synchroniseren. De repositoryconfiguratie sluit daarnaast lokale runtime-/buildartefacten uit (zoals `.artifacts` en testrapporten) die nooit mogen worden overgedragen. `.github/workflows/crabbox-hydrate.yml` beheert de checkout, de installatie van Node/pnpm, het ophalen van `origin/main` en de overdracht van de niet-geheime omgeving voor `crabbox run --id <cbx_id>`-opdrachten in de eigen cloud.

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [Ontwikkelingskanalen](/nl/install/development-channels)
