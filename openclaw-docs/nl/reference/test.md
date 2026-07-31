---
read_when:
    - Tests uitvoeren of repareren
summary: Tests lokaal uitvoeren (vitest) en wanneer je de force-/coveragemodi gebruikt
title: Tests
x-i18n:
    generated_at: "2026-07-27T06:33:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 391185703e853bb523e1396eb22da4693d10d47b1644d3b2a51707d329f67dae
    source_path: reference/test.md
    workflow: 16
---

- Volledige testkit (suites, live, Docker): [Testen](/nl/help/testing)
- Validatie van updates en pluginpakketten: [Updates en plugins testen](/nl/help/testing-updates-plugins)

## Standaardinstelling voor agents

Agentsessies voeren alleen lokaal één of enkele gerichte tests en goedkope statische controles uit
voor vertrouwde broncode en wanneer de bestaande installatie van afhankelijkheden gereed is. Voer
nooit lokaal tooling uit een niet-vertrouwde repository uit. Grotere suites, gewijzigde gates met
uitwaaierende typecheck-/linttaken, builds, Docker, pakkettaken, E2E, live bewijs en
platformoverschrijdende validatie worden extern via Crabbox uitgevoerd. Zwaar bewijs voor vertrouwde
maintainers wordt standaard via Blacksmith Testbox uitgevoerd. De geconfigureerde Testbox-workflow
laadt aanmeldgegevens, dus niet-vertrouwde code van bijdragers of forks moet in plaats daarvan
fork-CI zonder geheimen of een opgeschoonde directe AWS Crabbox gebruiken.

Warm niet vooraf op voor verwacht werk. Verkrijg de backend pas wanneer de
eerste zware opdracht gereed is, hergebruik de geretourneerde `tbx_...`-id voor latere zware
opdrachten, synchroniseer bij elke uitvoering de huidige checkout en stop deze vóór de overdracht.

Na het eerste succesvolle hergebruik registreert de wrapper de basis-,
afhankelijkheids- en Testbox-workflowvingerafdruk van de lease onder `.crabbox/testbox-leases/`.
Bij wijzigingen die alleen de broncode betreffen, wordt de opgewarmde box hergebruikt. Bij een gewijzigde samenvoegbasis, lockfile,
invoer voor de pakketbeheerder, wrapper of Testbox-workflow wordt veilig gestopt en is een
nieuwe lease vereist. Bij elke uitvoering wordt de huidige checkout nog steeds gesynchroniseerd.
`OPENCLAW_TESTBOX_ALLOW_STALE=1` is alleen bedoeld voor doelbewuste diagnostiek, niet
voor releasebewijs.

De onderstaande lokale testopdrachten zijn bedoeld voor menselijke workflows en begrensd agentbewijs.
Onbeschikbaarheid van een externe provider moet worden gemeld; dit geeft geen toestemming om
stilzwijgend een brede lokale gate uit te voeren.

Warm voor niet-vertrouwd zwaar bewijs pas op met `--provider aws`. Elke uitvoering moet
`CRABBOX_ENV_ALLOW=CI` instellen, `--provider aws --no-hydrate` doorgeven en
een nieuwe tijdelijke externe `HOME` gebruiken voordat afhankelijkheden worden geïnstalleerd of
tests worden uitgevoerd. Gebruik een nieuw opgewarmde lease die aan die niet-vertrouwde bron is toegewezen; hergebruik nooit
een vertrouwde of eerder met aanmeldgegevens geladen lease. Start een geïnstalleerd, vertrouwd Crabbox-
binair bestand vanuit een schone, vertrouwde `main`-checkout en haal alleen de externe PR op met
`--fresh-pr`; voer de wrapper of configuratie van de niet-vertrouwde checkout nooit lokaal uit.
Verwijder `CRABBOX_AWS_INSTANCE_PROFILE` uit de omgeving en stop veilig tenzij de herleide
`aws.instanceProfile` leeg is. Gebruik vóór elke installatie/test vertrouwde
tools met absolute paden om een IMDSv2-token te vereisen, aan te tonen dat het endpoint voor IAM-aanmeldgegevens
404 retourneert en te verifiëren dat de externe `git rev-parse HEAD` gelijk is aan de volledige
beoordeelde SHA van de PR-head. Koppel de lease aan die SHA en stop/warm opnieuw op wanneer de head
wijzigt. Upload de vertrouwde `scripts/crabbox-untrusted-bootstrap.sh` vanuit een schone
`main` naast `--fresh-pr`; deze installeert vastgepinde Node/pnpm, verifieert de SHA
en de pin van de pakketbeheerder, isoleert `HOME`, installeert afhankelijkheden en voert daarna
de aangevraagde test uit. Als de broker niet kan aantonen dat er geen rol is of als er geen externe PR bestaat,
gebruik dan fork-CI zonder geheimen. Gebruik geen `hydrate-github`, `--no-sync` of een
Testbox-workflow die met aanmeldgegevens is geladen.
Verwijder alle `CRABBOX_TAILSCALE*`-overschrijvingen uit de omgeving, dwing `--network public
--tailscale=false` af, wis exit-node-/LAN-vlaggen en vereis dat `crabbox inspect`
openbare netwerken zonder Tailscale-status rapporteert voordat een script wordt geüpload.

## Gebruikelijke lokale volgorde

1. `pnpm test:changed` voor Vitest-bewijs met gewijzigd bereik.
2. `pnpm test <path-or-filter>` voor één bestand, map of expliciet doel.
3. `pnpm test` alleen wanneer je doelbewust de volledige lokale Vitest-suite nodig hebt.

In een Codex-worktree of gekoppelde/sparse checkout vermijden agents rechtstreeks lokaal gebruik van
`pnpm test*` / `pnpm check*` / `pnpm crabbox:run`:

- Begrensd gericht bewijs met gereedstaande afhankelijkheden:
  `node scripts/run-vitest.mjs <path-or-filter>`.
- Controle met eerst classificatie van wijzigingen: `node scripts/check-changed.mjs`; plannen met alleen documentatie,
  zonder wijzigingen en met beperkte metadata blijven lokaal wanneer afhankelijkheden gereed zijn,
  terwijl zware plannen of plannen met ontbrekende afhankelijkheden aan Testbox worden gedelegeerd.
- Expliciet breed bewijs met behouden lease: `node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox ... -- env OPENCLAW_CHECK_CHANGED_REMOTE_CHILD=1 OPENCLAW_CHANGED_LANES_RAW_SYNC=1 corepack pnpm check:changed`, zodat pnpm binnen Testbox wordt uitgevoerd.
- De uiteindelijke `exitCode` en timing-JSON van de wrapper vormen het opdrachtresultaat. Een gedelegeerde uitvoering van Blacksmith GitHub Actions kan na een geslaagde SSH-opdracht `cancelled` tonen omdat de Testbox buiten de keepalive-actie wordt gestopt; controleer de samenvatting van de wrapper en de opdrachtuitvoer voordat je dit als een fout beschouwt.
- `OPENCLAW_HEAVY_CHECK_LOCK_SCOPE=worktree <local-heavy-check command>`: houdt de serialisatie van zware controles binnen de huidige worktree in plaats van de gemeenschappelijke Git-map voor opdrachten zoals `pnpm check:changed` en gerichte `pnpm test ...`. Gebruik dit alleen op lokale hosts met hoge capaciteit wanneer je doelbewust onafhankelijke controles in gekoppelde worktrees uitvoert.

## Kernopdrachten

Uitvoeringen van de testwrapper eindigen met een korte `[test] passed|failed|skipped ... in ...`-samenvatting; de eigen duurregel van Vitest blijft het detail per shard.

| Opdracht                                           | Wat deze doet                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test`                                       | Expliciete bestands-/mapdoelen worden via Vitest-taken met een beperkt bereik geleid. Uitvoeringen zonder doel vormen bewijs voor de volledige suite: vaste shardgroepen worden uitgebreid naar leaf-configuraties voor lokale parallelle uitvoering, waarbij de verwachte sharduitwaaiering vóór de start wordt weergegeven. De extensiegroep wordt altijd uitgebreid naar shardconfiguraties per extensie in plaats van één gigantisch hoofdprojectproces.           |
| `pnpm test:changed`                               | Goedkope, slimme uitvoering van gewijzigde tests: nauwkeurige doelen uit rechtstreekse testwijzigingen, aangrenzende `*.test.ts`-bestanden, expliciete brontoewijzingen en de lokale importgraaf. Brede configuratie-/pakketwijzigingen worden overgeslagen tenzij ze aan specifieke tests zijn gekoppeld.                                                                                                                               |
| `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` | Expliciete brede uitvoering van gewijzigde tests; gebruik dit wanneer een wijziging aan een testharnas, configuratie of pakket moet terugvallen op het bredere gedrag van Vitest voor gewijzigde tests.                                                                                                                                                                                                                        |
| `pnpm test:force`                                 | Maakt de geconfigureerde OpenClaw Gateway-poort vrij (standaard `18789`) en voert vervolgens de volledige suite uit met een geïsoleerde Gateway-poort, zodat servertests niet botsen met een actief exemplaar.                                                                                                                                                                                    |
| `pnpm test:coverage`                              | Genereert een informatief V8-dekkingsrapport voor de standaard unit-taak (`vitest.unit.config.ts`); er worden geen dekkingsdrempels afgedwongen.                                                                                                                                                                                                                             |
| `pnpm test:coverage:changed`                      | Alleen unit-dekking voor bestanden die sinds `origin/main` zijn gewijzigd.                                                                                                                                                                                                                                                                                                       |
| `pnpm changed:lanes`                              | Toont de architectuurtaken die worden geactiveerd door het verschil ten opzichte van `origin/main`.                                                                                                                                                                                                                                                                                      |
| `pnpm check:changed`                              | Classificeert de gewijzigde taken voordat de uitvoering wordt gekozen. Plannen met alleen documentatie, zonder wijzigingen en met beperkte metadata blijven lokaal wanneer afhankelijkheden gereed zijn; plannen met uitwaaierende typecheck-/linttaken, andere zware taken of ontbrekende lokale afhankelijkheden worden buiten CI aan Crabbox/Testbox gedelegeerd. Voert Vitest niet uit; gebruik `pnpm test:changed` of `pnpm test <target>` voor testbewijs. |

## Gedeelde teststatus en proceshelpers

- `src/test-utils/openclaw-test-state.ts`: gebruik dit vanuit Vitest wanneer een test een geïsoleerde `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, configuratiefixture, werkruimte, agentmap of opslag voor auth-profielen nodig heeft.
- `pnpm test:env-mutations:report`: niet-blokkerend rapport van tests/harnassen die `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, `OPENCLAW_WORKSPACE_DIR` of gerelateerde omgevingssleutels rechtstreeks wijzigen. Gebruik dit om migratiekandidaten voor de helper voor gedeelde teststatus te vinden.
- `test/helpers/openclaw-test-instance.ts`: E2E-tests op procesniveau die op één plek een actieve Gateway, CLI-omgeving, logregistratie en opschoning nodig hebben.
- Docker-/Bash-E2E-taken die `scripts/lib/docker-e2e-image.sh` sourcen, kunnen `docker_e2e_test_state_shell_b64 <label> <scenario>` aan de container doorgeven en deze decoderen met `scripts/lib/openclaw-e2e-instance.sh`; scripts met meerdere home-mappen kunnen `docker_e2e_test_state_function_b64` doorgeven en in elke flow `openclaw_test_state_create <label> <scenario>` aanroepen. `node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` schrijft een sourcebaar hostomgevingsbestand (de `--` vóór `create` voorkomt dat nieuwere Node-runtimes `--env-file` als een Node-vlag behandelen). Taken die een Gateway starten, kunnen `scripts/lib/openclaw-e2e-instance.sh` sourcen voor het herleiden van het entrypoint, het starten van een mock-OpenAI, starten op de voorgrond/achtergrond, gereedheidscontroles, export van de statusomgeving, logdumps en procesopschoning.

## Taken voor Control UI, TUI en extensies

- **Gemockte E2E voor de Control UI:** `pnpm test:ui:e2e` voert de Vitest + Playwright-lane uit die de Vite Control UI start en een echte Chromium-pagina aanstuurt via een gemockte Gateway-WebSocket. Tests staan in `ui/src/**/*.e2e.test.ts`; gedeelde mocks/besturingselementen staan in `ui/src/test-helpers/control-ui-e2e.ts`. `pnpm test:e2e` bevat deze lane. Agentruns gebruiken standaard Testbox/Crabbox, inclusief gerichte verificatie; gebruik `node scripts/run-vitest.mjs run --config test/vitest/vitest.ui-e2e.config.ts --configLoader runner ui/src/ui/e2e/chat-flow.e2e.test.ts` alleen voor een expliciete lokale fallback.
- **TUI-PTY-tests:** `node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts` voert de snelle PTY-lane met nepbackend uit. `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` of `pnpm tui:pty:test:watch --mode local` voert de tragere `tui --local`-smoketest uit, die alleen het externe modeleindpunt mockt. Controleer stabiele zichtbare tekst of fixture-aanroepen, niet onbewerkte ANSI-snapshots.
- `pnpm test:extensions` en `pnpm test extensions` voeren alle extensie-/Plugin-shards uit. Zware kanaalplugins, de browserplugin en OpenAI worden als afzonderlijke shards uitgevoerd; andere plugingroepen blijven gebundeld. `pnpm test extensions/<id>` voert één lane voor een gebundelde Plugin uit.
- Bronbestanden met tests in hetzelfde niveau worden eerst aan die tests gekoppeld, voordat wordt teruggevallen op bredere directoryglobs. Wijzigingen aan helpers onder `src/channels/plugins/contracts/test-helpers`, `src/plugin-sdk/test-helpers` en `src/plugins/contracts` gebruiken een lokale importgraaf om importerende tests uit te voeren in plaats van elke shard breed uit te voeren wanneer het afhankelijkheidspad precies is.
- Doelen voor contractdirectory's waaieren uit naar hun contractlanes: `pnpm test src/channels/plugins/contracts` voert de vier kanaalcontractconfiguraties uit en `pnpm test src/plugins/contracts` voert de configuratie voor Plugin-contracten uit, omdat de algemene `channels`-/`plugins`-projecten `contracts/**` uitsluiten.
- `auto-reply` wordt opgesplitst in drie afzonderlijke configuraties (`core`, `top-level`, `reply`), zodat de antwoordharness de lichtere tests op het hoogste niveau voor status, tokens en helpers niet domineert.
- Geselecteerde `plugin-sdk`- en `commands`-testbestanden worden via afzonderlijke lichte lanes geleid die alleen `test/setup.ts` behouden, terwijl runtime-intensieve gevallen op hun bestaande lanes blijven.
- De basisconfiguratie van Vitest gebruikt standaard `pool: "threads"` en `isolate: false`, waarbij de gedeelde niet-geïsoleerde runner in alle repoconfiguraties is ingeschakeld.
- `pnpm test:channels` voert `vitest.channels.config.ts` uit.

## Gateway en E2E

- Gateway-integratie is opt-in: `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` of `pnpm test:gateway`.
- `pnpm test:e2e`: geaggregeerde repo-E2E = `pnpm test:e2e:gateway && pnpm test:ui:e2e`.
- `pnpm test:e2e:gateway`: end-to-end-smoketests voor de Gateway (WS/HTTP/Node-koppeling met meerdere instanties). Gebruikt standaard `threads` + `isolate: false` met adaptieve workers in `vitest.e2e.config.ts`; stel af met `OPENCLAW_E2E_WORKERS=<n>`, uitgebreide logboeken met `OPENCLAW_E2E_VERBOSE=1`.
- `pnpm test:live`: live tests voor providers (Claude/Minimax/DeepSeek/z.ai/enz., afgeschermd door `*.live.test.ts`). Vereist API-sleutels en `LIVE=1` (of `OPENCLAW_LIVE_TEST=1`) om het overslaan uit te schakelen; uitgebreide uitvoer met `OPENCLAW_LIVE_TEST_QUIET=0`.

## Volledige Docker-suite (`pnpm test:docker:all`)

Bouwt de gedeelde live-testimage, verpakt OpenClaw eenmaal als een npm-tarball, bouwt/hergebruikt een kale Node-/Git-runnerimage plus een functionele image die die tarball in `/app` installeert, en voert vervolgens Docker-smoketestlanes uit via een gewogen scheduler. `scripts/package-openclaw-for-docker.mjs` is de enige lokale/CI-pakketverpakker en valideert de tarball plus `dist/postinstall-inventory.json` voordat Docker deze gebruikt.

- Kale image (`OPENCLAW_DOCKER_E2E_BARE_IMAGE`): lanes voor installatieprogramma, updates en Plugin-afhankelijkheden; koppelt de vooraf gebouwde tarball in plaats van gekopieerde repobronnen.
- Functionele image (`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`): lanes voor normale functionaliteit van de gebouwde app.
- Lanedefinities: `scripts/lib/docker-e2e-scenarios.mjs`. Planner: `scripts/lib/docker-e2e-plan.mjs`. Uitvoerder: `scripts/test-docker-all.mjs`.
- `node scripts/test-docker-all.mjs --plan-json` genereert het CI-plan dat door de scheduler wordt beheerd (lanes, imagetypen, behoeften aan pakketten/live-images, statusscenario's, controle van referenties) zonder Docker te bouwen of uit te voeren.

Planningsinstellingen (omgevingsvariabelen, standaardwaarden tussen haakjes):

| Omgevingsvariabele                                                                                              | Standaard           | Doel                                                                                                                                                                                                                                                                                       |
| --------------------------------------------------------------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`                                                                               | 10                  | Processlots.                                                                                                                                                                                                                                                                               |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM`                                                                          | 10                  | Providergevoelige tail-pool.                                                                                                                                                                                                                                                               |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`                                                                                | 9                   | Limiet voor zware live-providerlanes.                                                                                                                                                                                                                                                      |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`                                                                                 | 5                   | Limiet voor lanes met npm-resources.                                                                                                                                                                                                                                                       |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`                                                                             | 7                   | Limiet voor lanes met serviceresources.                                                                                                                                                                                                                                                    |
| `OPENCLAW_DOCKER_ALL_LIVE_CLAUDE_LIMIT` / `_CODEX_LIMIT` / `_GEMINI_LIMIT` / `_DROID_LIMIT` / `_OPENCODE_LIMIT` | 4                   | Limieten voor zware lanes per provider.                                                                                                                                                                                                                                                    |
| `OPENCLAW_DOCKER_ALL_LIVE_OPENAI_LIMIT` / `_TELEGRAM_LIMIT`                                                     | 1                   | Striktere limieten per provider.                                                                                                                                                                                                                                                           |
| `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` / `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`                                         | -                   | Overschrijving voor grotere hosts.                                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS`                                                                          | 2000                | Vertraging tussen het starten van lanes; voorkomt pieken bij het aanmaken door de lokale Docker-daemon.                                                                                                                                                                                    |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`                                                                           | 7,200,000 (120 min) | Fallbacktime-out per lane; geselecteerde live-/tail-lanes gebruiken strengere limieten.                                                                                                                                                                                                    |
| `OPENCLAW_DOCKER_ALL_LIVE_RETRIES`                                                                              | 1                   | Nieuwe pogingen bij tijdelijke storingen van live-providers.                                                                                                                                                                                                                               |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`                                                                                   | uit                 | Drukt het lanemanifest af zonder Docker uit te voeren.                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_STATUS_INTERVAL_MS`                                                                        | 30000               | Interval voor het afdrukken van de status van actieve lanes.                                                                                                                                                                                                                               |
| `OPENCLAW_DOCKER_ALL_TIMINGS`                                                                                   | aan                 | Hergebruik `.artifacts/docker-tests/lane-timings.json` voor sortering met de langste eerst; stel in op `0` om dit uit te schakelen.                                                                                                                                                                  |
| `OPENCLAW_DOCKER_ALL_LIVE_MODE`                                                                                 | -                   | `skip` voor alleen deterministische/lokale lanes, `only` voor alleen live-providerlanes. Aliassen: `pnpm test:docker:local:all`, `pnpm test:docker:live:all`. De modus met alleen live-lanes voegt de belangrijkste en laatste live-lanes samen in één pool met de langste eerst, zodat providerbuckets Claude-/Codex-/Gemini-werk samen groeperen. |
| `OPENCLAW_LIVE_CLI_BACKEND_SETUP_TIMEOUT_SECONDS`                                                               | 180                 | Time-out voor Docker-installatie van de CLI-backend.                                                                                                                                                                                                                                       |

Het patroon voor omgevingsvariabelen voor resourcelimieten is `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` (resourcenaam in hoofdletters, niet-alfanumerieke tekens samengevoegd tot `_`).

Overig gedrag: de runner voert standaard een preflightcontrole van Docker uit, ruimt verouderde OpenClaw E2E-containers op, deelt caches van CLI-tools van providers tussen compatibele lanes en stopt na de eerste fout met het plannen van nieuwe gepoolde lanes, tenzij `OPENCLAW_DOCKER_ALL_FAIL_FAST=0` is ingesteld. Als één lane de effectieve gewichts-/resourcelimiet op een host met lage parallelliteit overschrijdt, kan deze toch vanuit een lege pool starten en zelfstandig worden uitgevoerd totdat capaciteit wordt vrijgegeven. Logs per lane, `summary.json`, `failures.json` en fasetimings worden onder `.artifacts/docker-tests/<run-id>/` geschreven; gebruik `pnpm test:docker:timings <summary.json>` om trage lanes te inspecteren en `pnpm test:docker:rerun <run-id|summary.json|failures.json>` om eenvoudige gerichte opdrachten voor opnieuw uitvoeren weer te geven.

### Opmerkelijke Docker-lanes

| Opdracht                                                                    | Verifieert                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test:docker:browser-cdp-snapshot`                                     | Chromium-gebaseerde E2E-broncontainer met onbewerkte CDP + geïsoleerde Gateway; `browser doctor --deep` CDP-rolsnapshots bevatten link-URL's, door de cursor gepromoveerde klikbare elementen, iframe-verwijzingen en framemetadata.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `pnpm test:docker:skill-install`                                            | Installeert de ingepakte tarball in een kale Docker-runner met `skills.install.allowUploadedArchives: false`, haalt via een live ClawHub-zoekopdracht een huidige skill-slug op, installeert deze via `openclaw skills install` en verifieert `SKILL.md`, `.clawhub/origin.json`, `.clawhub/lock.json` en `skills info --json`.                                                                                                                                                                                                                                                                                                                                                                                                            |
| `pnpm test:docker:live-cli-backend:claude`, `:claude:resume`, `:claude:mcp` | Gerichte live-probes voor de CLI-backend; Gemini heeft overeenkomstige aliassen `:resume` en `:mcp`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `pnpm test:docker:openwebui`                                                | OpenClaw + Open WebUI in Docker: aanmelden, `/api/models` controleren en een echte geproxiede chat uitvoeren via `/api/chat/completions`. Vereist een bruikbare sleutel voor een live model en haalt een externe image op; er wordt niet verwacht dat dit even CI-stabiel is als de unit-/E2E-suites.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `pnpm test:docker:mcp-channels`                                             | Vooraf gevulde Gateway-container plus een clientcontainer die `openclaw mcp serve` start: gerouteerde gespreksdetectie, het lezen van transcripten, metagegevens van bijlagen, gedrag van de live-gebeurteniswachtrij, routering van uitgaande verzendingen en kanaal- en machtigingsmeldingen in Claude-stijl via de echte stdio-bridge (de assertie leest onbewerkte stdio-MCP-frames rechtstreeks).                                                                                                                                                                                                                                                                                                                                                         |
| `pnpm test:docker:upgrade-survivor`                                         | Installeert de ingepakte tarball over een vervuilde fixture van een oude gebruiker, voert een pakketupdate plus niet-interactieve doctor uit zonder live sleutels voor providers/kanalen, start een loopback-Gateway en controleert of agents/kanaalconfiguratie/allowlists voor plugins/werkruimte-/sessiebestanden/verouderde afhankelijkheidsstatus van legacy-plugins/opstart-/RPC-status behouden blijven.                                                                                                                                                                                                                                                                                                                                           |
| `pnpm test:docker:published-upgrade-survivor`                               | Installeert standaard `openclaw@latest`, vult realistische bestanden van bestaande gebruikers vooraf in, configureert via een ingebakken `openclaw config set`-recept, werkt bij naar de ingepakte tarball, voert niet-interactieve doctor uit, schrijft `.artifacts/upgrade-survivor/summary.json` en controleert `/healthz`, `/readyz` en de RPC-status. Overschrijf met `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`, breid een matrix uit met `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` of voeg scenariofixtures toe met `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues` (bevat `configured-plugin-installs` en `stale-source-plugin-shadow`). Package Acceptance maakt deze beschikbaar als `published_upgrade_survivor_baseline(s)` / `_scenarios` en verwerkt metatokens zoals `last-stable-4` of `all-since-2026.4.23`. |
| `pnpm test:docker:update-migration`                                         | Testharnas voor overleving na een gepubliceerde upgrade in het scenario `plugin-deps-cleanup`, dat standaard begint bij `openclaw@2026.4.23`. De workflow `Update Migration` breidt dit uit met `baselines=all-since-2026.4.23` om buiten Full Release CI het opschonen van afhankelijkheden van geconfigureerde plugins aan te tonen.                                                                                                                                                                                                                                                                                                                                                                                               |
| `pnpm test:docker:plugins`                                                  | Installatie-/updatesmoketest voor een lokaal pad, `file:`, npm-registerpakketten met gehoste afhankelijkheden, bewegende git-verwijzingen, ClawHub-fixtures, marketplace-updates en het inschakelen/inspecteren van de Claude-bundel.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

## Lokale PR-gate

Voer voor lokale controles voor het landen/de gate van een PR het volgende uit:

- `pnpm check:changed`
- `pnpm check`
- `pnpm check:test-types`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

Als `pnpm test` onregelmatig faalt op een zwaar belaste host, voer deze dan eenmaal opnieuw uit voordat je dit als een regressie beschouwt en isoleer vervolgens met `pnpm test <path/to/test>`. Voor hosts met beperkt geheugen:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## Tools voor testprestaties

- `pnpm test:perf:imports`: schakelt rapportage van Vitest-importduur + importuitsplitsing in, terwijl voor expliciete bestands-/mapdoelen nog steeds routering via lanes met een beperkt bereik wordt gebruikt. `pnpm test:perf:imports:changed` beperkt dezelfde profilering tot bestanden die sinds `origin/main` zijn gewijzigd.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` vergelijkt de prestaties van het gerouteerde pad in gewijzigde modus met de native uitvoering van het rootproject voor dezelfde gecommitte git-diff; `pnpm test:perf:changed:bench -- --worktree` benchmarkt de huidige wijzigingenset in de worktree zonder deze eerst te committen.
- `pnpm test:perf:profile:main` schrijft een CPU-profiel voor de hoofdthread van Vitest (`.artifacts/vitest-main-profile`); `pnpm test:perf:profile:runner` schrijft CPU- en heap-profielen voor de unit-runner (`.artifacts/vitest-runner-profile`).
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json`: voert elke Vitest-leafconfiguratie van de volledige suite serieel uit en schrijft gegroepeerde duurgegevens plus JSON-/logartefacten per configuratie. Rapporten voor de volledige suite isoleren standaard bestanden, zodat behouden modulegraphs en GC-pauzes van eerdere bestanden niet worden toegerekend aan latere asserties; geef `-- --no-isolate` alleen door wanneer je doelbewust de accumulatie in een gedeelde worker profileert. De Test Performance Agent gebruikt dit als basislijn voordat wordt geprobeerd trage tests te verbeteren. `pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json` vergelijkt gegroepeerde rapporten na een prestatiegerichte wijziging.
- Uitvoeringen van volledige suites, extensieshards en shards met opnamepatronen werken lokale timinggegevens in `.artifacts/vitest-shard-timings.json` bij; latere uitvoeringen van volledige configuraties gebruiken die timings om trage en snelle shards evenwichtig te verdelen. CI-shards met opnamepatronen voegen de shardnaam aan de timingsleutel toe, zodat timings van gefilterde shards zichtbaar blijven zonder timinggegevens van de volledige configuratie te vervangen. Stel `OPENCLAW_TEST_PROJECTS_TIMINGS=0` in om het lokale timingartefact te negeren.

## Benchmarks

<Accordion title="Modellatentie (scripts/bench-model.ts)">

```bash
pnpm tsx scripts/bench-model.ts --runs 10
```

Optionele omgevingsvariabelen: `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`. Standaardprompt: "Antwoord met één woord: ok. Geen interpunctie of extra tekst."

</Accordion>

<Accordion title="CLI-opstarttijd (scripts/bench-cli-startup.ts)">

```bash
pnpm test:startup:bench
pnpm test:startup:bench:smoke
pnpm test:startup:bench:save
pnpm test:startup:bench:update
pnpm test:startup:bench:check
pnpm tsx scripts/bench-cli-startup.ts --runs 12
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3
pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all
```

Voorinstellingen:

- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `tasks --json`, `tasks list --json`, `tasks audit --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all`: beide voorinstellingen gecombineerd

De uitvoer bevat `sampleCount`, gemiddelde, p50, p95, minimum/maximum, de verdeling van afsluitcodes/signalen en het maximale RSS per opdracht. `--cpu-prof-dir` / `--heap-prof-dir` schrijven V8-profielen per uitvoering.

Opgeslagen uitvoer: `pnpm test:startup:bench:smoke` schrijft `.artifacts/cli-startup-bench-smoke.json`; `pnpm test:startup:bench:save` schrijft `.artifacts/cli-startup-bench-all.json` (`runs=5 warmup=1`). Ingecheckte fixture: `test/fixtures/cli-startup-bench.json`, vernieuwd door `pnpm test:startup:bench:update`, vergeleken door `pnpm test:startup:bench:check`.

</Accordion>

<Accordion title="Opstarten van de Gateway (scripts/bench-gateway-startup.ts)">

Gebruikt standaard het gebouwde CLI-ingangspunt op `dist/entry.js`; voer eerst `pnpm build` uit. Geef `--entry scripts/run-node.mjs` door om in plaats daarvan de bronrunner te meten en houd die resultaten gescheiden van de basiswaarden voor het gebouwde ingangspunt.

```bash
pnpm test:startup:gateway -- --runs 5 --warmup 1
pnpm test:startup:gateway -- --case skipChannels --case fiftyPlugins --runs 5
node --import tsx scripts/bench-gateway-startup.ts --case default --runs 5 --output .artifacts/gateway-startup.json
```

Case-id's: `default`, `skipChannels` (opstarten van kanalen overgeslagen), `oneInternalHook`, `allInternalHooks`, `fiftyPlugins` (50 manifestplugins), `fiftyStartupLazyPlugins` (50 manifestplugins die bij het opstarten lui worden geladen).

De uitvoer bevat de eerste procesuitvoer, `/healthz`, `/readyz`, de logtijd voor het luisteren via HTTP, de logtijd waarop de Gateway gereed is, CPU-tijd, de verhouding van CPU-kernen, maximaal RSS, heap, metrische gegevens van de opstarttracering, vertraging van de eventloop en gedetailleerde metrische gegevens van de opzoektabel voor plugins. Het script stelt `OPENCLAW_GATEWAY_STARTUP_TRACE=1` in de omgeving van de onderliggende Gateway in.

`/healthz` is levendigheid (de HTTP-server kan antwoorden). `/readyz` is bruikbare gereedheid (sidecars van opstartplugins, kanalen en gereedheidskritiek werk na het koppelen zijn afgerond). Opstarthooks worden asynchroon aangeroepen en maken geen deel uit van de gereedheidsgarantie. De logtijd voor gereedheid is de interne tijdstempel van de Gateway, nuttig voor toeschrijving aan de proceszijde, maar geen vervanging voor de externe `/readyz`-probe.

Gebruik JSON-uitvoer of `--output` bij het vergelijken van wijzigingen. Gebruik `--cpu-prof-dir` alleen nadat traceringsuitvoer wijst op import-, compileer- of CPU-gebonden werk dat niet alleen met fasetimings kan worden verklaard.

</Accordion>

<Accordion title="Herstarten van de Gateway (scripts/bench-gateway-restart.ts)">

Alleen macOS en Linux (gebruikt SIGUSR1 voor herstarts binnen het proces; mislukt onmiddellijk op Windows). Dezelfde standaard voor het gebouwde ingangspunt en `--entry scripts/run-node.mjs`-overschrijving als bij het opstarten van de Gateway hierboven.

```bash
pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5
pnpm test:restart:gateway -- --case default --runs 3 --restarts 3 --warmup 1
```

Case-id's: `skipChannels`, `skipChannelsAcpxProbe` (ACPX-opstartprobe ingeschakeld), `skipChannelsNoAcpxProbe` (probe uitgeschakeld), `default`, `fiftyPlugins`.

De uitvoer bevat de volgende `/healthz`, de volgende `/readyz`, uitvaltijd, gereedheidstiming van de herstart, CPU, RSS, metrische gegevens van de opstarttracering voor het vervangende proces en metrische gegevens van de herstarttracering voor signaalafhandeling, het leegmaken van actief werk, afsluitfasen, de volgende start, gereedheidstiming en geheugensnapshots. Het script stelt `OPENCLAW_GATEWAY_STARTUP_TRACE=1` en `OPENCLAW_GATEWAY_RESTART_TRACE=1` in.

Gebruik deze benchmark wanneer een wijziging invloed heeft op herstartsignalering, afsluithandlers, opstarten na een herstart, afsluiten van sidecars, overdracht van services of gereedheid na een herstart. Begin met `skipChannels` om de werking van de Gateway te isoleren van het opstarten van kanalen; gebruik `default` of cases met veel plugins pas nadat de beperkte case het herstartpad verklaart. Traceringsgegevens zijn aanwijzingen voor toeschrijving, geen definitieve conclusies — beoordeel een herstartwijziging aan de hand van meerdere steekproeven, de overeenkomende owner-span, het gedrag van `/healthz`/`/readyz` en het voor de gebruiker zichtbare herstartcontract.

</Accordion>

## E2E voor onboarding (Docker)

Optioneel; alleen nodig voor gecontaineriseerde smoke-tests van onboarding. Volledige cold-startflow in een schone Linux-container:

```bash
scripts/e2e/onboard-docker.sh
```

Doorloopt de interactieve wizard via een pseudo-tty, verifieert configuratie-, werkruimte- en sessiebestanden, start vervolgens de Gateway en voert `openclaw health` uit.

## Smoke-test voor QR-import (Docker)

Garandeert dat de onderhouden QR-runtimehelper wordt geladen onder de ondersteunde Docker Node-runtimes (standaard Node 24, compatibel met Node 22):

```bash
pnpm test:docker:qr
```

## Gerelateerd

- [Testen](/nl/help/testing)
- [Live testen](/nl/help/testing-live)
- [Updates en plugins testen](/nl/help/testing-updates-plugins)
