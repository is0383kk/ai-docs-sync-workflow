---
read_when:
    - Gedrag voor OpenClaw-updates, doctor, pakketacceptatie of Plugin-installatie wijzigen
    - Een releasecandidate voorbereiden of goedkeuren
    - Fouten opsporen bij pakketupdates, het opschonen van Plugin-afhankelijkheden of regressies bij de installatie van Plugins
sidebarTitle: Update and plugin tests
summary: Hoe OpenClaw updatepaden, pakketmigraties en installatie- en updategedrag van plugins valideert
title: 'Testen: updates en plugins'
x-i18n:
    generated_at: "2026-07-27T05:36:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96a11fe42472f758d4fd1cc568486e301f7460982fdb547cab8b39de04a8dabe
    source_path: help/testing-updates-plugins.md
    workflow: 16
---

Checklist voor update- en Plugin-validatie: bewijs dat het installeerbare pakket
echte gebruikersstatus kan bijwerken, verouderde legacy-status kan herstellen via `doctor`, en nog steeds
plugins uit elke ondersteunde bron kan installeren, laden, bijwerken en verwijderen.

Zie [Testen](/nl/help/testing) voor het bredere overzicht van testrunners. Zie voor live providersleutels
en suites die het netwerk gebruiken [Live testen](/nl/help/testing-live).

## Wat we beschermen

- Een pakkettarball is volledig, heeft een geldige `dist/postinstall-inventory.json`,
  en is niet afhankelijk van uitgepakte repobestanden.
- Een gebruiker kan van een ouder gepubliceerd pakket naar het kandidaatpakket overstappen
  zonder configuratie, agents, sessies, werkruimten, toestemmingslijsten voor plugins of
  kanaalconfiguratie te verliezen.
- `openclaw doctor --fix --non-interactive` beheert paden voor het opschonen en herstellen van
  legacy-status. Bij het opstarten mogen geen verborgen compatibiliteitsmigraties voor verouderde
  Plugin-status ontstaan.
- Plugin-installaties werken vanuit lokale mappen, git-repo's, npm-pakketten en het
  registerpad van ClawHub.
- npm-afhankelijkheden van plugins worden geïnstalleerd in één beheerd npm-project per Plugin,
  vóór vertrouwen gescand en tijdens het verwijderen van de Plugin via `npm uninstall`
  verwijderd, zodat gehoiste afhankelijkheden niet achterblijven.
- Een Plugin-update doet niets wanneer er niets is gewijzigd: installatierecords, de opgeloste
  bron, de indeling van geïnstalleerde afhankelijkheden en de ingeschakelde status blijven intact.

## Lokaal bewijs tijdens ontwikkeling

Begin gericht:

```bash
pnpm changed:lanes --json
pnpm check:changed
pnpm test:changed
```

Voer bij wijzigingen aan Plugin-installatie, -verwijdering, afhankelijkheden of pakketinventaris ook
de gerichte tests uit die de bewerkte overgang afdekken:

```bash
pnpm test src/plugins/uninstall.test.ts src/infra/package-dist-inventory.test.ts test/scripts/package-acceptance-workflow.test.ts
```

Bewijs het pakketartefact voordat een Docker-pakkettraject een tarball gebruikt:

```bash
pnpm release:check
```

`release:check` voert controles op afwijkingen tussen configuratie, documentatie en API uit (configuratieschema, basislijn voor configuratiedocumentatie,
API-contractmanifest en exports van de Plugin-SDK, Plugin-versies/-inventaris),
schrijft de distributie-inventaris van het pakket, voert `npm pack --dry-run` uit, weigert verboden
ingepakte bestanden, installeert de tarball onder een tijdelijk voorvoegsel, voert postinstall uit en
voert rooktests uit op de toegangspunten van gebundelde kanalen.

## Docker-trajecten

De Docker-trajecten vormen het bewijs op productniveau. Ze installeren een echt pakket of werken dit bij
in Linux-containers en controleren het gedrag via CLI-opdrachten,
het opstarten van de Gateway, HTTP-probes, RPC-status en bestandssysteemstatus.

Gebruik gerichte trajecten tijdens het itereren:

```bash
pnpm test:docker:plugins
pnpm test:docker:plugin-lifecycle-matrix
pnpm test:docker:plugin-update
pnpm test:docker:upgrade-survivor
pnpm test:docker:published-upgrade-survivor
pnpm test:docker:update-restart-auth
pnpm test:docker:update-migration
```

Belangrijke trajecten:

- `test:docker:plugins` omvat rooktests voor Plugin-installatie, installaties uit lokale mappen,
  gedrag waarbij updates van lokale mappen worden overgeslagen, lokale mappen met vooraf geïnstalleerde
  afhankelijkheden, installaties van `file:`-pakketten, git-installaties met CLI-uitvoering, git-updates
  van verplaatsende referenties, installaties uit het npm-register met gehoiste transitieve
  afhankelijkheden, npm-updates die niets doen, weigering van ongeldige npm-pakketmetadata,
  installaties van lokale ClawHub-fixtures en updates die niets doen, updategedrag voor de marketplace
  en het inschakelen/inspecteren van de Claude-bundel. Stel `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` in om
  het ClawHub-blok hermetisch/offline te houden.
- `test:docker:plugin-lifecycle-matrix` installeert het kandidaatpakket in een kale
  container en doorloopt voor een npm-Plugin installatie, inspectie, uitschakeling, inschakeling,
  expliciete upgrade, expliciete downgrade en verwijdering nadat de Plugin-code is verwijderd.
  Het registreert RSS- en CPU-metrieken per fase.
- `test:docker:plugin-update` valideert dat een ongewijzigde geïnstalleerde Plugin
  tijdens `openclaw plugins update` niet opnieuw wordt geïnstalleerd en geen installatiemetadata verliest.
- `test:docker:upgrade-survivor` installeert de kandidaat-tarball over een vervuilde
  fixture van een oude gebruiker, voert een pakketupdate plus niet-interactieve doctor uit, start vervolgens
  een loopback-Gateway en controleert het behoud van status.
- `test:docker:published-upgrade-survivor` installeert eerst een gepubliceerde basislijn,
  configureert deze via een ingebakken `openclaw config set`-recept, werkt deze bij naar de
  kandidaat-tarball, voert doctor uit, controleert het opschonen van legacy-status, start de Gateway en
  peilt `/healthz`, `/readyz` en de RPC-status.
- `test:docker:update-restart-auth` installeert het kandidaatpakket, start een
  beheerde Gateway met tokenauthenticatie, verwijdert de omgevingsvariabele voor Gateway-authenticatie van de aanroeper voor
  `openclaw update --yes --json` en vereist dat de updateopdracht van de kandidaat
  de Gateway opnieuw start vóór de normale probes.
- `test:docker:update-migration` is het opschoningsintensieve traject voor gepubliceerde updates. Het
  begint met een geconfigureerde gebruikersstatus in Discord/Telegram-stijl, voert doctor van de basislijn uit
  zodat geconfigureerde Plugin-afhankelijkheden de kans krijgen om te worden aangemaakt, voegt
  legacy-resten van Plugin-afhankelijkheden toe voor een geconfigureerde verpakte Plugin, werkt bij naar
  de kandidaat-tarball en vereist dat doctor na de update de legacy-hoofdmappen van
  afhankelijkheden verwijdert.

Nuttige varianten voor overleving van gepubliceerde upgrades:

```bash
OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@2026.4.23 \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=versioned-runtime-deps \
pnpm test:docker:published-upgrade-survivor

OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@latest \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=bootstrap-persona \
pnpm test:docker:published-upgrade-survivor
```

Beschikbare scenario's: `base`, `acpx-openclaw-tools-bridge`, `feishu-channel`,
`bootstrap-persona`, `channel-post-core-restore`, `plugin-deps-cleanup`,
`configured-plugin-installs`, `stale-source-plugin-shadow`, `tilde-log-path`
en `versioned-runtime-deps`. Bij geaggregeerde uitvoeringen wordt `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues`
(alias `far-reaching`) uitgebreid naar alle scenario's, inclusief de
installatiemigratie voor geconfigureerde plugins.

Volledige updatemigratie staat bewust los van volledige release-CI. Gebruik de
handmatige `Update Migration`-workflow wanneer de releasevraag luidt: "kan elke
gepubliceerde stabiele release vanaf 2026.4.23 naar deze kandidaat worden bijgewerkt en
resten van Plugin-afhankelijkheden opschonen?":

```bash
gh workflow run update-migration.yml \
  --ref main \
  -f workflow_ref=main \
  -f package_ref=main \
  -f baselines=all-since-2026.4.23 \
  -f scenarios=plugin-deps-cleanup
```

## Pakketacceptatie

Pakketacceptatie is de systeemeigen GitHub-pakketpoort. Deze zet één kandidaatpakket
om in een `package-under-test`-tarball, registreert versie en SHA-256 en
voert vervolgens herbruikbare Docker-E2E-trajecten uit tegen exact die tarball. De workflowharnasreferentie
staat los van de bronreferentie van het pakket, zodat de huidige testlogica oudere
vertrouwde releases kan valideren.

Kandidaatbronnen:

- `source=npm`: valideer `openclaw@extended-stable`, `openclaw@beta`,
  `openclaw@latest` of een exact gepubliceerde versie.
- `source=ref`: pak een vertrouwde branch, tag of commit in met het geselecteerde huidige
  harnas.
- `source=url`: valideer een openbare HTTPS-tarball met vereiste `package_sha256`.
  Dit pad weigert URL-referenties, niet-standaard HTTPS-poorten, privé/interne
  hostnamen of DNS/IP-resultaten, IP-ruimte voor speciaal gebruik en onveilige omleidingen.
- `source=trusted-url`: valideer een HTTPS-tarball met vereiste
  `package_sha256` en `trusted_source_id` tegen het beleid in beheer van de maintainers
  in `.github/package-trusted-sources.json`. Gebruik dit voor zakelijke/privé-
  mirrors in plaats van `source=url` af te zwakken met een invoerschakelaar die privétoegang toestaat.
  Bearer-authenticatie gebruikt, wanneer door beleid geconfigureerd, het vaste
  `OPENCLAW_TRUSTED_PACKAGE_TOKEN`-secret.
- `source=artifact`: hergebruik een tarball die door een andere Actions-uitvoering is geüpload.

Volledige releasevalidatie gebruikt standaard `source=artifact`, gebouwd vanuit de
opgeloste release-SHA. Geef voor bewijs na publicatie
`package_acceptance_package_spec=openclaw@YYYY.M.PATCH` door, zodat dezelfde upgradematrix
zich in plaats daarvan op het uitgebrachte npm-pakket richt.

Releasecontroles roepen Pakketacceptatie aan met de set voor pakket/update/herstart/Plugin:

```text
doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape
```

Wanneer release-soak is ingeschakeld (verplicht voor `release_profile=stable` en
`full`), geven ze ook het volgende door:

```text
published_upgrade_survivor_baselines=last-stable-4 2026.4.23 2026.5.2 2026.4.15
published_upgrade_survivor_scenarios=reported-issues
telegram_mode=mock-openai
```

Hierdoor blijven pakketmigratie, wisselen van updatekanaal, tolerantie voor beschadigde beheerde plugins,
opschoning van verouderde Plugin-afhankelijkheden, offline Plugin-dekking, Plugin-
updategedrag en Telegram-pakket-QA op hetzelfde opgeloste artefact, zonder
de standaardpoort voor releasepakketten elke gepubliceerde release te laten doorlopen.

`last-stable-4` wordt omgezet naar de vier recentste stabiele, op npm gepubliceerde OpenClaw-
releases. Releasepakketacceptatie legt `2026.4.23` vast als de eerste compatibiliteitsgrens
voor Plugin-updates, `2026.5.2` als grens voor wijzigingen in de Plugin-architectuur en
`2026.4.15` als een oudere basislijn voor gepubliceerde updates uit 2026.4.1x; de resolver
verwijdert dubbele vastgelegde versies die al tot de recentste vier behoren. Gebruik voor volledige dekking van
migratie van gepubliceerde updates `all-since-2026.4.23` in de afzonderlijke workflow voor
updatemigratie in plaats van volledige release-CI. `release-history` blijft
beschikbaar voor handmatige, bredere steekproeven wanneer je ook het legacy-anker
van vóór de datum wilt meenemen.

Wanneer meerdere basislijnen voor overleving van gepubliceerde upgrades zijn geselecteerd, verdeelt de herbruikbare
Docker-workflow elke basislijn over een eigen gerichte runner-taak. Elke
basislijnshard voert nog steeds de geselecteerde scenarioset uit, maar logboeken en artefacten blijven
per basislijn gescheiden en de doorlooptijd wordt begrensd door de langzaamste shard in plaats van door één grote
seriële taak.

Voer handmatig een pakketprofiel uit wanneer je vóór de release een kandidaat valideert:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=package \
  -f published_upgrade_survivor_baselines="last-stable-4 2026.4.23 2026.5.2 2026.4.15" \
  -f published_upgrade_survivor_scenarios=reported-issues \
  -f telegram_mode=mock-openai
```

Stel voor een gepubliceerde extended-stable-canary
`package_spec=openclaw@extended-stable` in. Pakketacceptatie zet die
selector om in een exacte tarball voordat de Docker-trajecten worden uitgevoerd.

Gebruik `suite_profile=product` wanneer de releasevraag MCP-kanalen,
opschoning van Cron/subagents, OpenAI-zoekopdrachten op het web of OpenWebUI omvat. Gebruik `suite_profile=full`
alleen wanneer je volledige Docker-dekking van het releasepad nodig hebt.

## Standaard voor releases

Voor releasekandidaten is de standaard bewijsstack:

1. `pnpm check:changed` en `pnpm test:changed` voor regressies op bronniveau.
2. `pnpm release:check` voor de integriteit van pakketartefacten.
3. Het Pakketacceptatieprofiel `package` of de aangepaste pakkettrajecten van releasecontroles
   voor installatie-/update-/herstart-/Plugin-contracten.
4. Releasecontroles voor meerdere besturingssystemen voor besturingssysteemspecifieke installatieprogramma's, onboarding en platformgedrag.
5. Live suites alleen wanneer het gewijzigde oppervlak het gedrag van providers of gehoste services
   raakt.

Op machines van maintainers moeten brede poorten en Docker-/pakketbewijs op productniveau
in Testbox worden uitgevoerd, tenzij expliciet lokaal bewijs wordt verzameld.

## Legacy-compatibiliteit

Compatibiliteitstolerantie is beperkt en tijdgebonden:

- Pakketten tot en met `2026.4.25`, waaronder `2026.4.25-beta.*`, mogen
  al uitgebrachte hiaten in pakketmetadata in Pakketacceptatie tolereren.
- Het gepubliceerde `2026.4.26`-pakket mag waarschuwen voor reeds uitgebrachte
  lokale bestanden met stempels voor buildmetadata.
- Latere pakketten moeten aan moderne contracten voldoen. Dezelfde hiaten leiden tot fouten in plaats van
  waarschuwingen of overslaan.

Voeg voor deze oude vormen geen nieuwe opstartmigraties toe. Voeg een doctor-
herstel toe of breid dit uit en bewijs het vervolgens met `upgrade-survivor`, `published-upgrade-survivor` of
`update-restart-auth` wanneer de updateopdracht de herstart beheert.

## Dekking toevoegen

Voeg bij wijzigingen in update- of plugingedrag dekking toe op de laagste laag die
om de juiste reden kan mislukken:

- Zuivere pad- of metadatalogica: unit-test naast de broncode.
- Pakketinventaris of gedrag van ingepakte bestanden: `package-dist-inventory` of test voor
  tarballcontrole.
- CLI-installatie-/updategedrag: assertie of fixture in de Docker-lane.
- Migratiegedrag van gepubliceerde releases: `published-upgrade-survivor`-scenario.
- Herstartgedrag onder beheer van de update: `update-restart-auth`.
- Gedrag van register-/pakketbronnen: `test:docker:plugins`-fixture of ClawHub-
  fixtureserver.
- Gedrag van afhankelijkheidsindeling of opschoning: controleer zowel de uitvoering tijdens runtime als de
  bestandssysteemgrens. npm-afhankelijkheden kunnen binnen het beheerde npm-project
  van de plugin op een hoger niveau worden geplaatst. Tests moeten daarom aantonen dat dit project wordt gescand/opgeschoond,
  in plaats van ervan uit te gaan dat alleen de lokale `node_modules`-structuur van het pluginpakket wordt gebruikt.

Houd nieuwe Docker-fixtures standaard hermetisch. Gebruik lokale fixtureregisters en
neppakketten, tenzij live registergedrag het doel van de test is.

## Fouttriage

Begin met de identiteit van het artefact:

- Samenvatting van Package Acceptance `resolve_package`: bron, versie, SHA-256 en
  artefactnaam.
- Docker-artefacten: `.artifacts/docker-tests/**/summary.json`,
  `failures.json`, lanelogboeken en opdrachten voor opnieuw uitvoeren.
- Samenvatting van overlevende upgrades: `.artifacts/upgrade-survivor/summary.json`,
  inclusief basisversie, kandidaatversie, scenario, fasetijden en
  dekking van configuratierecepten.

Geef de voorkeur aan het opnieuw uitvoeren van precies de mislukte lane met hetzelfde pakketartefact boven
het opnieuw uitvoeren van de volledige overkoepelende release.
