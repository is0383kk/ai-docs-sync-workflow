---
doc-schema-version: 1
read_when:
    - Volledige releasevalidatie uitvoeren of opnieuw uitvoeren
    - Vergelijking van validatieprofielen voor stabiele en volledige releases
    - Fouten in fasen van releasevalidatie opsporen
summary: Fasen van volledige releasevalidatie, onderliggende workflows, releaseprofielen, handles voor opnieuw uitvoeren en bewijs
title: Volledige releasevalidatie
x-i18n:
    generated_at: "2026-07-27T05:32:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf165d5515f4b9bb11d239382649d332d20bb8a32bd4492ae99092fb5ee2216
    source_path: reference/full-release-validation.md
    workflow: 16
---

`Full Release Validation` is de overkoepelende productvalidatie voor releases. Het meeste werk
vindt plaats in onderliggende workflows, zodat een mislukte omgeving opnieuw kan worden uitgevoerd zonder de
volledige release opnieuw te starten. Voer de releasevoorbereiding uit voordat je de Code SHA vastlegt; deze
vernieuwt de locale-uitvoer van de Control UI wanneer de achtergrondbot deze nog niet heeft
opgeleverd en dwingt vervolgens dezelfde strikte controle zonder fallbacks af die door release-CI wordt gebruikt.

Leg de productvolledige commit van vóór de changelog vast als de **Code SHA** en voer vervolgens uit:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

`provider` accepteert ook `anthropic` of `minimax` voor onboarding op meerdere besturingssystemen en de
end-to-end agentbeurt. De helper leidt het profiel `beta` af uit alfa-/bèta-
pakketversies en anders `stable`. Geef alternatieve workflowinvoer door met
`-f key=value`; gebruik `-f release_profile=full` alleen voor de brede adviescontrole.

De helper maakt een tijdelijke `release-ci/*`-ref die is vastgezet op één vertrouwde
`origin/main`-workflow-SHA, geeft de doel-SHA alleen door als de kandidaat-`ref`
en verwijdert de tijdelijke ref na validatie. Elke gestarte onderliggende workflow moet
dezelfde workflow-SHA rapporteren. Geef
`-f reuse_evidence=false` door om een nieuwe uitvoering af te dwingen of
`--workflow-sha <trusted-main-sha>` om een oudere workflowcommit te selecteren die nog
bereikbaar is vanaf de huidige `origin/main`. De workflow maakt of wijzigt zelf nooit
repositoryrefs.

## Uitzondering voor extended-stable

Voor publicatie van extended-stable is een uitvoering vereist waarvan zowel de workflow als het doel de
canonieke branch is:

```bash
gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

Gebruik `pnpm ci:full-release` of `release-ci/*` niet. De publicatie koppelt de
branch, head-/doel-SHA, manifest-`workflowRef`, ID en poging van de uitvoering aan de canonieke
branch en releasecommit.

Backport productfouten; voer voor tooling met een vastgelegd doel de kleinste reparatie uit die
het gedrag behoudt; probeer provider-, goedkeurings- of runnerfouten opnieuw zonder
bronwijziging. Elke branchwijziging vereist een volledig nieuwe uitvoering. Laat vereiste
pakket-, installatieprogramma-, update-, kanaal- of livefunctionaliteit niet weg omdat het doel oud is.

Wanneer de Code SHA voor een reguliere release groen is, genereer en commit je alleen
`CHANGELOG.md`. Deze nieuwe commit is de **Release SHA**. Voer dezelfde helper uit voor
de Release SHA. Productbewijs wordt alleen hergebruikt wanneer GitHub bewijst dat de Release
SHA afstamt van de Code SHA en de volledige verzameling gewijzigde paden exact
`CHANGELOG.md` is; de npm-preflight en acceptatie van pakket/installatie worden nog steeds uitgevoerd op de
Release SHA.

`release_profile=stable` en `release_profile=full` voeren altijd de uitgebreide
live-/Docker-duurtest uit. Geef `run_release_soak=true` door om dezelfde duurtestlanes
op te nemen met het profiel `beta`. Stabiele publicatie weigert een validatiemanifest
zonder deze duurtest en blokkerend bewijs van productprestaties.

Package Acceptance bouwt het kandidaattarball normaal vanuit de opgeloste
`ref`, inclusief uitvoeringen met volledige SHA die met `pnpm ci:full-release` zijn gestart. Geef na een
bètapublicatie `release_package_spec=openclaw@YYYY.M.PATCH-beta.N` door om
het uitgebrachte npm-pakket te hergebruiken voor releasecontroles, Package Acceptance, meerdere besturingssystemen,
het Docker-releasepad en pakket-Telegram. Gebruik `package_acceptance_package_spec`
alleen wanneer Package Acceptance bewust een ander pakket moet aantonen.
De livepakketlane van de Codex-plugin volgt dezelfde status: gepubliceerde
`release_package_spec`-waarden leiden `codex_plugin_spec=npm:@openclaw/codex@<version>` af;
SHA-/artifactuitvoeringen verpakken `extensions/codex` vanuit de geselecteerde ref; en operators
kunnen `codex_plugin_spec` rechtstreeks instellen voor `npm:`-, `npm-pack:`- of `git:`-pluginbronnen.
De lane verleent de expliciete goedkeuring voor installatie van de Codex CLI die deze plugin vereist,
en voert vervolgens een Codex CLI-preflight en OpenAI-agentbeurten in dezelfde sessie uit.
De laatste beurt zonder nieuwe pogingen en met gemiddeld denkniveau verstuurt zichtbare voortgang met weggelaten
Codex-`final`, leest willekeurige invoer uit de werkruimte, schrijft het exacte artifact ervan
en verstuurt een expliciete voltooiing. Hiermee wordt de regressie in v2026.7.1 gedetecteerd waarbij het
versturen van gewone voortgang de beurt beëindigde.

## Fasen op hoofdniveau

Voor `rerun_group=all` wordt eerst een `Check for reusable validation evidence`-job
uitgevoerd. Deze zoekt naar de nieuwste eerdere groene volledige validatie met hetzelfde release-
profiel, dezelfde effectieve duurtestinstelling en dezelfde validatie-invoer. Nieuwe uitvoeringen met exact hetzelfde doel gebruiken
`exact-target-full-validation-v1`. Een afstammeling waarvan de volledige delta exact
`CHANGELOG.md` is, gebruikt `changelog-only-release-v1`; elke productlane wordt overgeslagen
en de verificator controleert onafhankelijk opnieuw de GitHub-commitvergelijking, het onveranderlijke
bovenliggende artifact, de onderliggende uitvoeringen en de dispatchlogboeken. Elke andere doelwijziging vereist
een nieuwe Code SHA-validatie. Geef `reuse_evidence=false` door om een nieuwe volledige
uitvoering af te dwingen. Bewijshergebruik wordt alleen uitgevoerd vanuit `main` of een canonieke, op SHA vastgezette
`release-ci/*`-ref waarvan de workflowcommit deel blijft uitmaken van de vertrouwde `main`-afstammingslijn;
andere workflowrefs voeren de geselecteerde lanes opnieuw uit.

Nieuwe pakketgerichte validatie bereidt één onveranderlijk tarball en één Docker-
imageartifact voor voordat Plugin Prerelease en OpenClaw Release Checks worden gestart.
Beide onderliggende workflows verifiëren vóór gebruik dezelfde pakket-SHA, artifact-ID's, servicedigests,
poging van de producerende uitvoering en digest van het Docker-archief. De pakketonafhankelijke
basis-Dockerlaag gebruikt een inhoudsgeadresseerde GHCR-cache; kandidaatspecifieke images
blijven onveranderlijke GitHub-artifacts. Gerichte uitvoeringen met een expliciete specificatie van een gepubliceerd
pakket behouden in plaats daarvan het bestaande pakketpad.

Ook voor `rerun_group=all` bouwt een `Verify Docker runtime image assets`-job
het Docker-doel `runtime-assets` met
`OPENCLAW_EXTENSIONS=diagnostics-otel,codex`. Deze wordt parallel met de
andere fasen uitgevoerd en door de overkoepelende verificator afgedwongen; lanes wachten er niet langer op
voordat ze worden gestart. Een beperktere `rerun_group` slaat deze preflight over.

| Fase                    | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Doelbepaling            | **Job:** `Resolve target ref`<br />**Onderliggende workflow:** geen<br />**Toont aan:** bepaalt de releasebranch, tag of volledige commit-SHA en registreert de geselecteerde invoer.<br />**Opnieuw uitvoeren:** voer de overkoepelende workflow opnieuw uit als dit mislukt.                                                                                                                                                                                                                                                                                                            |
| Gedeelde kandidaat      | **Job:** `Prepare shared release candidate`<br />**Onderliggende workflow:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Toont aan:** verpakt en valideert één pakket met exacte SHA, bouwt één functionele Docker-image en registreert onveranderlijke tuples van pakket- en imageartifacts voor beide pakketgerichte onderliggende workflows.<br />**Opnieuw uitvoeren:** voer de betrokken pakket-, plugin-prerelease-, multi-OS- of live-/E2E-groep opnieuw uit.                                                                                                                 |
| Preflight van Docker-assets | **Job:** `Verify Docker runtime image assets`<br />**Onderliggende workflow:** geen<br />**Toont aan:** het Docker-builddoel `runtime-assets` slaagt nog steeds voordat een andere fase wordt gestart. Wordt alleen uitgevoerd voor `rerun_group=all`.<br />**Opnieuw uitvoeren:** voer de overkoepelende workflow opnieuw uit met `rerun_group=all`.                                                                                                                                                                                                                                         |
| Vitest en normale CI    | **Job:** `Run normal full CI`<br />**Onderliggende workflow:** `CI`<br />**Toont aan:** handmatige volledige CI-grafiek voor de doelref, inclusief Linux Node-lanes, shards voor gebundelde plugins, shards voor plugin- en kanaalcontracten, compatibiliteit met Node 22, `check-*`, `check-additional-*`, smokecontroles van gebouwde artifacts, documentatiecontroles, Python-Skills, Windows, macOS, Control UI-i18n en Android via de overkoepelende workflow.<br />**Opnieuw uitvoeren:** `rerun_group=ci`.                                                                                          |
| Plugin-prerelease       | **Job:** `Run plugin prerelease validation`<br />**Onderliggende workflow:** `Plugin Prerelease`<br />**Toont aan:** statische plugincontroles uitsluitend voor releases, agentische plugindekking, volledige batchshards voor plugins, Docker-lanes voor plugin-prereleases en een niet-blokkerend `plugin-inspector-advisory`-artifact voor compatibiliteitstriage.<br />**Opnieuw uitvoeren:** `rerun_group=plugin-prerelease`.                                                                                                                                                          |
| Releasecontroles        | **Job:** `Run release/live/Docker/QA validation`<br />**Onderliggende workflow:** `OpenClaw Release Checks`<br />**Toont aan:** installatiesmoke, pakketcontroles op meerdere besturingssystemen, Package Acceptance, pariteit van QA Lab, live Matrix en Telegram, plus afgeschermde adviserende lanes voor Discord, WhatsApp en Slack. Stabiele en volledige profielen voeren ook uitgebreide live-/E2E-suites en onderdelen van het Docker-releasepad uit; bèta kan zich hiervoor aanmelden met `run_release_soak=true`.<br />**Opnieuw uitvoeren:** `rerun_group=release-checks` of een beperktere handle voor releasecontroles.              |
| Pakket-Telegram         | **Job:** `Run package Telegram E2E`<br />**Onderliggende workflow:** `NPM Telegram Beta E2E`<br />**Toont aan:** een gerichte Telegram-E2E voor een gepubliceerd pakket wanneer `release_package_spec` of `npm_telegram_package_spec` is ingesteld. Volledige kandidaatvalidatie gebruikt in plaats daarvan de canonieke Telegram-E2E van Package Acceptance.<br />**Opnieuw uitvoeren:** `rerun_group=npm-telegram` met `release_package_spec` of `npm_telegram_package_spec`.                                                                                                              |
| Productprestaties       | **Job:** `Run product performance evidence`<br />**Onderliggende workflow:** `OpenClaw Performance`<br />**Toont aan:** prestatie-uitvoering voor het releaseprofiel (`profile=release`, `repeat=3`, `fail_on_regression=true`, `publish_reports=false`) voor de doel-SHA. Kova-uitvoer blijft in workflowartifacts en de onderliggende workflow moet aantonen dat de rapportpublicator is overgeslagen. Alleen vereist (blokkerend) voor `rerun_group=all` of `rerun_group=performance`; niet vereist voor beperktere groepen die opnieuw worden uitgevoerd.<br />**Opnieuw uitvoeren:** `rerun_group=performance`. |
| Overkoepelende verificator | **Job:** `Verify full validation`<br />**Onderliggende workflow:** geen<br />**Toont aan:** controleert de geregistreerde conclusies van onderliggende uitvoeringen opnieuw en voegt tabellen met de langzaamste jobs uit onderliggende workflows toe.<br />**Opnieuw uitvoeren:** voer alleen deze job opnieuw uit nadat een mislukte onderliggende workflow opnieuw is uitgevoerd en groen is.                                                                                                                                                                                                                                                                 |

De overkoepelende workflow start productprestaties altijd in de modus met alleen artifacts.
`OpenClaw Performance` staat rapportpublicatie alleen toe voor geplande uitvoeringen of een
handmatige start waarbij `publish_reports=true` expliciet is ingesteld. De beveiliging voor
alleen artifacts moet succesvol worden voltooid en daarmee aantonen dat de publicatorjob overgeslagen bleef.
Nieuw en hergebruikt bewijs registreert
`controls.performanceReportPublication=artifact-only`; de verificator en selector voor hergebruik
weigeren bewijs zonder het overeenkomende genormaliseerde bewijs van de onderliggende prestatieworkflow.

De verifier uploadt het canonieke manifest als
`full-release-validation-<run-id>-<run-attempt>`. De tooling voor bewijsmateriaal valideert
de artefact-ID, digest, producerende run en poging voordat exact die
artefact-ID wordt gedownload. De tooling begrenst het gedownloade ZIP-bestand, verifieert de bytes ervan aan de hand van de REST-
`sha256:`-digest en streamt de enige toegestane begrensde manifestvermelding zonder
het archief uit te pakken. Een alias met een stabiele naam blijft tijdelijk bestaan voor oudere
publicatieconsumenten. De verifier geeft altijd de voorkeur aan het aan de poging gekoppelde artefact;
als overgang accepteert deze de stabiele naam alleen voor een poging-1-producent van manifest v2.
De verifier weigert die verouderde naam voor latere pogingen en manifest v3.

Voor `ref=main` met `rerun_group=all`, voor `release/*`-refs en voor Tideclaw-
alfarefs vervangt een nieuwere overkoepelende run een oudere met dezelfde ref en
herstartgroep. Wanneer de bovenliggende run wordt geannuleerd, annuleert de monitor ervan elke onderliggende
workflow die al is gestart. Validatieruns voor tags en vastgezette SHA's
annuleren elkaar niet.

## Fasen van releasecontroles

`OpenClaw Release Checks` is de grootste onderliggende workflow. Deze bepaalt het doel
eenmalig en valideert het gedeelde pakketartefact van de overkoepelende workflow wanneer dit beschikbaar is. Een
directe of gerichte dispatch bereidt een eigen `release-package-under-test`-
artefact voor wanneer pakket- of Docker-gerichte fasen dit nodig hebben.

| Fase                     | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Releasedoel              | **Taak:** `Resolve target ref`<br />**Onderliggende workflow:** geen<br />**Tests:** geselecteerde ref, optionele verwachte SHA, profiel, herstartgroep en filter voor een gerichte livesuite.<br />**Opnieuw uitvoeren:** `rerun_group=release-checks`.                                                                                                                                                                                                                                                                                                                                                             |
| Pakketartefact           | **Taak:** `Prepare release package artifact`<br />**Onderliggende workflow:** geen<br />**Tests:** valideert de onveranderlijke pakkettuple van de overkoepelende workflow, of verpakt één kandidaat-tarball voor een directe/gerichte dispatch van Releasecontroles en stelt deze vervolgens beschikbaar aan navolgende pakketgerichte controles.<br />**Opnieuw uitvoeren:** de betreffende pakket-, cross-OS- of live/E2E-groep.                                                                                                                                                                                                                                |
| Installatierooktest      | **Taak:** `Run install smoke`<br />**Onderliggende workflow:** `Install Smoke`<br />**Tests:** volledig installatiepad met hergebruik van de rooktestimage van het Dockerfile in de hoofdmap, installatie van het QR-pakket, Docker-rooktests voor de hoofdmap en Gateway, Docker-tests voor het installatieprogramma en een imageprovider-rooktest voor globale installatie met Bun.<br />**Opnieuw uitvoeren:** `rerun_group=install-smoke`.                                                                                                                                                                                                                                                           |
| Cross-OS                 | **Taak:** `cross_os_release_checks`<br />**Onderliggende workflow:** `OpenClaw Cross-OS Release Checks (Reusable)`<br />**Tests:** trajecten voor nieuwe installatie en upgrade op Linux, Windows en macOS voor de geselecteerde provider en modus, met de kandidaat-tarball plus een basislijnpakket.<br />**Opnieuw uitvoeren:** `rerun_group=cross-os`.                                                                                                                                                                                                                                                                 |
| Repo- en live-E2E        | **Taak:** `Run repo/live E2E validation`<br />**Onderliggende workflow:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Tests:** E2E voor de repository, livecache, OpenAI-websocketstreaming, native liveprovider- en Plugin-shards en door Docker ondersteunde harnesses voor livemodellen/backends/Gateway, geselecteerd door `release_profile`.<br />**Runs:** `run_release_soak=true`, `release_profile=full` of gericht `rerun_group=live-e2e`.<br />**Opnieuw uitvoeren:** `rerun_group=live-e2e`, optioneel met `live_suite_filter`.                                                                                |
| Docker-releasepad        | **Taak:** `Run Docker release-path validation`<br />**Onderliggende workflow:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Tests:** Docker-chunks voor het releasepad met het gedeelde pakketartefact.<br />**Runs:** `run_release_soak=true`, `release_profile=full` of gericht `rerun_group=live-e2e`.<br />**Opnieuw uitvoeren:** `rerun_group=live-e2e`.                                                                                                                                                                                                                                     |
| Pakketacceptatie         | **Taak:** `Run package acceptance`<br />**Onderliggende workflow:** `Package Acceptance`<br />**Tests:** offline pakketfixtures voor Plugins, Plugin-update, de canonieke E2E van het mock-OpenAI Telegram-pakket en controles van overlevende gepubliceerde upgrades met dezelfde tarball. Blokkerende releasecontroles gebruiken standaard de laatst gepubliceerde basislijn; duurcontroles (`run_release_soak=true`) worden uitgebreid naar de laatste 4 stabiele npm-releases plus 3 vastgezette historische versies (`2026.4.23`, `2026.5.2`, `2026.4.15`) en worden uitgevoerd met upgradefixtures voor gemelde problemen.<br />**Opnieuw uitvoeren:** `rerun_group=package`. |
| Volwassenheidsscorekaart | **Taak:** `Render maturity scorecard release docs`<br />**Onderliggende workflow:** `maturity-scorecard.yml`<br />**Tests:** rendert de adviserende documentatie van de volwassenheidsscorekaart voor de doel-ref. Wordt alleen uitgevoerd wanneer `run_maturity_scorecard=true` wordt doorgegeven.<br />**Opnieuw uitvoeren:** `rerun_group=qa` met `run_maturity_scorecard=true`.                                                                                                                                                                                                                                                           |
| QA-pariteit              | **Taak:** `Run QA Lab parity lane` en `Run QA Lab parity report`<br />**Onderliggende workflow:** directe taken<br />**Tests:** agentische pariteitspakketten voor kandidaat en basislijn, gevolgd door het pariteitsrapport.<br />**Opnieuw uitvoeren:** `rerun_group=qa-parity` of `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                         |
| QA-runtimepariteit       | **Taak:** `Verify QA Lab runtime-pair lanes`<br />**Onderliggende workflow:** directe taak<br />**Tests:** het canonieke kerntraject `openclaw`/`codex` (`pnpm openclaw qa suite --runtime-pair openclaw,codex --runtime-pair-lane core`) en, met `run_release_soak=true`, het duurtraject. Adviserend: afzonderlijke trajecttaken blokkeren de verifier voor releasecontroles niet.<br />**Opnieuw uitvoeren:** `rerun_group=qa-parity` of `rerun_group=qa`.                                                                                                                                                             |
| QA-dekking van runtimetools | **Taak:** `Enforce QA Lab runtime tool coverage`<br />**Onderliggende workflow:** directe taak<br />**Tests:** dynamische toolafwijking tussen `openclaw` en `codex` in het canonieke kerntraject voor runtimeparen (`pnpm openclaw qa coverage --tools`), waarbij de uitvoer van dat traject wordt gebruikt. Blokkerend: deze taak kan niet adviserend worden overschreven.<br />**Opnieuw uitvoeren:** `rerun_group=qa-parity` of `rerun_group=qa`.                                                                                                                                                                                                     |
| QA live Matrix           | **Taak:** `Run QA Live Matrix profile`<br />**Onderliggende workflow:** herbruikbare workflow `QA-Lab - All Lanes`<br />**Tests:** door pariteit bewezen YAML-scenario's via de gedeelde liveadapter voor Matrix in de `qa-live-shared`-omgeving.<br />**Opnieuw uitvoeren:** `rerun_group=qa-live` of `rerun_group=qa`; gebruik `live_suite_filter=qa-live-matrix` voor een gerichte herstart van Matrix.                                                                                                                                                                                                                    |
| QA live Telegram         | **Taak:** `Run QA Lab live Telegram lane`<br />**Onderliggende workflow:** vertrouwde dispatch `OpenClaw Release Telegram QA`<br />**Tests:** live-QA voor Telegram met leases voor Convex-CI-referenties.<br />**Opnieuw uitvoeren:** `rerun_group=qa-live` of `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                                 |
| QA live Discord          | **Taak:** `Run QA Lab live Discord lane`<br />**Onderliggende workflow:** directe adviserende taak<br />**Tests:** live-QA voor Discord met leases voor Convex-CI-referenties wanneer `OPENCLAW_RELEASE_QA_DISCORD_LIVE_CI_ENABLED` is ingeschakeld.<br />**Opnieuw uitvoeren:** `rerun_group=qa-live` met `live_suite_filter=qa-live-discord`.                                                                                                                                                                                                                                                                            |
| QA live WhatsApp         | **Taak:** `Run QA Lab live WhatsApp lane`<br />**Onderliggende workflow:** directe adviserende taak<br />**Tests:** live-QA voor WhatsApp met leases voor Convex-CI-referenties wanneer `OPENCLAW_RELEASE_QA_WHATSAPP_LIVE_CI_ENABLED` is ingeschakeld.<br />**Opnieuw uitvoeren:** `rerun_group=qa-live` met `live_suite_filter=qa-live-whatsapp`.                                                                                                                                                                                                                                                                        |
| QA live Slack            | **Taak:** `Run QA Lab live Slack lane`<br />**Onderliggende workflow:** directe adviserende taak<br />**Tests:** live-QA voor Slack met leases voor Convex-CI-referenties wanneer `OPENCLAW_RELEASE_QA_SLACK_LIVE_CI_ENABLED` is ingeschakeld.<br />**Opnieuw uitvoeren:** `rerun_group=qa-live` met `live_suite_filter=qa-live-slack`.                                                                                                                                                                                                                                                                                    |
| Releaseverifier          | **Taak:** `Verify release checks`<br />**Onderliggende workflow:** geen<br />**Tests:** vereiste releasecontroletaken voor de geselecteerde herstartgroep.<br />**Opnieuw uitvoeren:** opnieuw uitvoeren nadat de gerichte onderliggende taken zijn geslaagd.                                                                                                                                                                                                                                                                                                                                                                                   |

## Docker-chunks voor het releasepad

De Docker-releasepadfase voert deze chunks uit wanneer `live_suite_filter`
leeg is:

| Chunk                                                           | Dekking                                                                                                                                      |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `core`                                                          | Kern-smokelanes voor het Docker-releasepad.                                                                                                  |
| `package-update-openai`                                         | Installatie- en updategedrag van het OpenAI-pakket, Codex-installatie op aanvraag, opvolging van live voortgang van de Codex-plugin en toolaanroepen voor Chat Completions. |
| `package-update-anthropic`                                      | Installatie- en updategedrag van het Anthropic-pakket.                                                                                       |
| `package-update-core`                                           | Providerneutraal pakket- en updategedrag.                                                                                                    |
| `plugins-runtime-plugins`                                       | Plugin-runtimelanes die Plugin-gedrag testen.                                                                                                |
| `plugins-runtime-services`                                      | Door services ondersteunde en live Plugin-runtimelanes.                                                                                      |
| `plugins-runtime-install-a` tot en met `plugins-runtime-install-h` | Installatie-/runtimebatches van Plugins, opgesplitst voor parallelle releasevalidatie.                                                        |
| `openwebui`                                                     | OpenWebUI-compatibiliteitssmoke, indien aangevraagd geïsoleerd op een speciale runner met een grote schijf.                                  |

Gebruik gerichte `docker_lanes=<lane[,lane]>` in de herbruikbare live/E2E-workflow wanneer
slechts één Docker-lane is mislukt. De releaseartefacten bevatten per lane
opdrachten voor opnieuw uitvoeren, met invoer voor hergebruik van pakketartefacten en images indien beschikbaar.

## Releaseprofielen

`release_profile` bepaalt voornamelijk de breedte van live/providers binnen releasecontroles.
Het verwijdert geen normale volledige CI, Plugin Prerelease, installatiesmoke, pakketacceptatie
of QA Lab. Stabiele en volledige profielen voeren altijd uitputtende repo/live-
E2E- en Docker-releasepad-soakdekking uit. Het bètaprofiel kan dit inschakelen met
`run_release_soak=true`. Package Acceptance levert de canonieke pakket-
Telegram-E2E voor elke volledige kandidaat, zodat de overkoepelende workflow die
live-poller niet dupliceert.

| Profiel  | Beoogd gebruik                     | Inbegrepen live-/providerdekking                                                                                                                                                                           |
| -------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `beta`   | Snelste releasekritieke smoke.    | Live-pad voor OpenAI/core, live Docker-modellen voor OpenAI, native Gateway-core, native OpenAI Gateway-profiel, native OpenAI-Plugin en live Docker-Gateway voor OpenAI.                                  |
| `stable` | Standaardprofiel voor releasegoedkeuring. | `beta` plus Anthropic-smoke, Google, MiniMax, backend, native live-testharnas, live Docker-CLI-backend, Docker ACP-bind, Docker Codex-harnas, Docker-subagent-aankondiging en een OpenCode Go-smokeshard. |
| `full`   | Brede adviserende controle.       | `stable` plus adviserende providers, live Plugin-shards en live mediashards.                                                                                                                      |

## Toevoegingen alleen voor volledig

Deze suites worden overgeslagen door `stable` en opgenomen door `full`:

| Gebied                           | Dekking alleen voor volledig                                                                                             |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Live Docker-modellen             | OpenCode Go, OpenRouter, xAI, Z.ai en Fireworks.                                                                         |
| Live Docker-Gateway              | Adviserende providers opgesplitst in shards voor DeepSeek/Fireworks, OpenCode Go/OpenRouter en xAI/Z.ai.                 |
| Providerprofielen voor native Gateway | Volledige Anthropic Opus- en Sonnet/Haiku-shards, Fireworks, DeepSeek, volledige OpenCode Go-modelshards, OpenRouter, xAI en Z.ai. |
| Live shards voor native Plugins  | Plugins A-K, L-N, overige O-Z, Moonshot en xAI.                                                                          |
| Live shards voor native media    | Audio, Google Music, MiniMax Music en videogroepen A-D.                                                                  |

`stable` omvat `native-live-src-gateway-profiles-anthropic-smoke` en
`native-live-src-gateway-profiles-opencode-go-smoke`; `full` gebruikt in plaats daarvan de bredere
modelshards voor Anthropic en OpenCode Go. Gerichte nieuwe uitvoeringen kunnen nog steeds de
geaggregeerde handles `native-live-src-gateway-profiles-anthropic` of
`native-live-src-gateway-profiles-opencode-go` gebruiken.

## Gerichte nieuwe uitvoeringen

Gebruik `rerun_group` om te voorkomen dat niet-gerelateerde releaseboxen opnieuw worden uitgevoerd:

| Handle              | Bereik                                                                                          |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `all`               | Alle fasen van Full Release Validation.                                                         |
| `ci`                | Alleen de handmatige volledige CI-child.                                                        |
| `plugin-prerelease` | Alleen de Plugin Prerelease-child.                                                              |
| `release-checks`    | Alle fasen van OpenClaw Release Checks.                                                         |
| `install-smoke`     | Install Smoke tot en met releasecontroles.                                                      |
| `cross-os`          | Releasecontroles voor meerdere besturingssystemen.                                              |
| `live-e2e`          | Repo/live-E2E- en Docker-releasepadvalidatie.                                                    |
| `package`           | Package Acceptance.                                                                            |
| `qa`                | QA-pariteit plus live QA-lanes.                                                                 |
| `qa-parity`         | Alleen QA-pariteitslanes en rapport.                                                            |
| `qa-live`           | Live QA-lanes voor Matrix/Telegram plus afgeschermde Discord-, WhatsApp- en Slack-lanes indien ingeschakeld. |
| `npm-telegram`      | Telegram-E2E voor het gepubliceerde pakket; vereist `release_package_spec` of `npm_telegram_package_spec`. |
| `performance`       | Alleen bewijs van productprestaties.                                                            |

Gebruik `live_suite_filter` met `rerun_group=live-e2e` wanneer één live suite is mislukt.
Geldige filter-id's zijn gedefinieerd in de herbruikbare live/E2E-workflow, waaronder
`docker-live-models`, `live-gateway-docker`,
`live-gateway-anthropic-docker`, `live-gateway-google-docker`,
`live-gateway-minimax-docker`, `live-gateway-advisory-docker`,
`live-cli-backend-docker`, `live-acp-bind-docker` en
`live-codex-harness-docker`.

Stel voor een gerichte nieuwe uitvoering van een QA-transport `rerun_group=qa-live` in en gebruik de
canonieke selector `qa-live-matrix`, `qa-live-telegram`, `qa-live-discord`,
`qa-live-whatsapp` of `qa-live-slack`.

De handle `live-gateway-advisory-docker` is een geaggregeerde handle voor het opnieuw uitvoeren van de
drie providershards en vertakt daarom nog steeds naar alle adviserende Docker-Gateway-taken.

Gebruik `cross_os_suite_filter` met `rerun_group=cross-os` wanneer één lane voor meerdere besturingssystemen
is mislukt. Het filter accepteert een besturingssysteem-id, een suite-id of een combinatie van besturingssysteem/suite, bijvoorbeeld
`windows/packaged-upgrade`, `windows` of `packaged-fresh`. Samenvattingen voor meerdere besturingssystemen
bevatten timings per fase voor verpakte upgradelanes, en langlopende
opdrachten drukken Heartbeat-regels af, zodat een vastgelopen update zichtbaar is vóór de
taaktime-out.

Mislukte QA-releasecontroles blokkeren normale releasevalidatie alleen voor geselecteerde
lanes voor Matrix, Telegram en dekking van QA-runtimetools. QA-pariteit, runtime-
pariteit en de afgeschermde live lanes voor Discord, WhatsApp en Slack zijn adviserend en
publiceren statusartefacten zonder de releaseverificatie te blokkeren. Tideclaw-
alphauitvoeringen kunnen releasecontroles die niet over pakketveiligheid gaan nog steeds als adviserend behandelen. Met
`release_profile=beta` zijn de live-providersuites van `Run repo/live E2E validation`
adviserend: implementaties van modellen van derden veranderen tijdens een release, dus
beta geeft hun fouten weer als waarschuwingen, terwijl stabiele en volledige profielen
ze blokkerend houden. Wanneer
`live_suite_filter` expliciet een afgeschermde live QA-lane aanvraagt, zoals Discord,
WhatsApp of Slack, moet de bijbehorende repo-
variabele `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` zijn ingeschakeld; anders mislukt het vastleggen van de invoer in plaats van de lane stilzwijgend over te slaan.
Voer `rerun_group=qa`, `qa-parity` of `qa-live` opnieuw uit wanneer je
nieuw QA-bewijs nodig hebt.

## Te bewaren bewijs

Bewaar de samenvatting `Full Release Validation` als index op releaseniveau. Deze koppelt
child-uitvoerings-id's en bevat tabellen met de langzaamste taken. Inspecteer bij fouten eerst de child-
workflow en voer vervolgens de kleinste overeenkomende handle hierboven opnieuw uit.

Leg voor een reguliere release zowel de Code SHA als de Release SHA vast, evenals het hergebruikbeleid
en de set gewijzigde paden, de groene parent-uitvoering van de Code SHA en de lichtgewicht parent-
uitvoering van de Release SHA. Leg voor extended-stable de canonieke branch, de exacte release-
SHA, de nieuwe parent-uitvoerings-id en poging, de workflowreferentie, elke child-uitvoering en eventuele
compatibiliteitsreparaties voor het bevroren doel of bewuste weglatingen vast.

Nuttige artefacten:

- `release-package-under-test` uit `OpenClaw Release Checks`
- Docker-releasepadartefacten onder `.artifacts/docker-tests/`
- Package Acceptance `package-under-test` en Docker-acceptatieartefacten
- Artefacten van releasecontroles voor meerdere besturingssystemen voor elk besturingssysteem en elke suite
- QA-pariteit, runtimepariteit en geselecteerde artefacten voor Matrix, Telegram, Discord, WhatsApp
  of Slack

## Workflowbestanden

- `.github/workflows/full-release-validation.yml`
- `.github/workflows/openclaw-release-checks.yml`
- `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml`
- `.github/workflows/plugin-prerelease.yml`
- `.github/workflows/install-smoke.yml`
- `.github/workflows/install-smoke-reusable.yml`
- `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`
- `.github/workflows/package-acceptance.yml`
- `.github/workflows/openclaw-performance.yml`
- `.github/workflows/npm-telegram-beta-e2e.yml`
