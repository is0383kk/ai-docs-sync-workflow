---
read_when:
    - Tests lokaal of in CI uitvoeren
    - Regressietests toevoegen voor bugs in modellen/providers
    - Gateway- en agentgedrag debuggen
summary: 'Testkit: unit-/e2e-/live-testsuites, Docker-runners en wat elke test behandelt'
title: Testen
x-i18n:
    generated_at: "2026-07-27T05:48:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw heeft drie Vitest-suites (unit/integratie, e2e, live) plus Docker-
runners. Deze pagina beschrijft wat elke suite omvat, welke opdracht je voor een
bepaalde workflow uitvoert, hoe live tests referenties ontdekken en hoe je
regressietests toevoegt voor echte bugs in providers/modellen.

<Note>
**QA-stack (qa-lab, qa-channel, live transportlanes)** wordt afzonderlijk gedocumenteerd:

- [QA-overzicht](/nl/concepts/qa-e2e-automation) - architectuur, opdrachtinterface, scenario-ontwikkeling en Matrix-profielen.
- [Volwassenheidsscorekaart](/nl/maturity/scorecard) - hoe QA-bewijs voor releases beslissingen over stabiliteit en LTS ondersteunt.
- [QA-kanaal](/nl/channels/qa-channel) - de synthetische transportplugin die wordt gebruikt door scenario's die door de repository worden ondersteund.

Deze pagina behandelt de reguliere testsuites en Docker/Parallels-runners. [QA-specifieke runners](#qa-specific-runners) hieronder vermeldt de concrete `qa`-aanroepen en verwijst terug naar de bovenstaande referenties.
</Note>

## Snel aan de slag

Op de meeste dagen:

- Volledige gate (verwacht vóór pushen): `pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- Snellere lokale uitvoering van de volledige suite op een ruime machine: `pnpm test:max`
- Directe Vitest-watchlus: `pnpm test:watch`
- Directe bestandsselectie routeert ook plugin-/kanaalpaden: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Geef bij het itereren op één fout eerst de voorkeur aan gerichte uitvoeringen.
- Door Docker ondersteunde QA-site: `pnpm qa:lab:up`
- Door een Linux-VM ondersteunde QA-lane: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Wanneer je tests aanpast of extra zekerheid wilt:

- Informatief V8-dekkingsrapport: `pnpm test:coverage`
- E2E-suite: `pnpm test:e2e`

## Tijdelijke testmappen

Gebruik de gedeelde helpers in `test/helpers/temp-dir.ts` voor tijdelijke mappen
die eigendom zijn van tests, zodat het eigenaarschap expliciet is en het
opruimen onderdeel blijft van de levenscyclus van de test:

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("gebruikt een tijdelijke werkruimte", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // gebruik de werkruimte
});
```

`useAutoCleanupTempDirTracker(afterEach)` stelt bewust geen handmatige
opruimmethode beschikbaar - Vitest beheert het opruimen na elke test. Oudere
helpers op een lager niveau (`makeTempDir`, `cleanupTempDirs`, `createTempDirTracker`) bestaan nog
voor tests die niet zijn gemigreerd; vermijd nieuw gebruik ervan en vermijd nieuwe losse
`fs.mkdtemp*`-aanroepen, tenzij een test expliciet onbewerkte werking van
tijdelijke mappen verifieert. Wanneer een losse tijdelijke map echt nodig is, voeg je een
controleerbare toestemmingsopmerking met een reden toe:

```ts
// openclaw-temp-dir: allow verifieert onbewerkte opruimwerking van het bestandssysteem
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs` rapporteert nieuwe aanmaak van losse
tijdelijke mappen en nieuw handmatig gebruik van gedeelde helpers in toegevoegde diff-regels,
zonder bestaande opruimstijlen te blokkeren. Het volgt dezelfde classificatie van testpaden
als `scripts/changed-lanes.mjs` en slaat de implementatie van de gedeelde helper
zelf over. `check:changed` voert dit rapport voor gewijzigde testpaden uit als een
CI-signaal met alleen waarschuwingen (GitHub-waarschuwingsannotaties, geen fouten).

## Live- en Docker/Parallels-workflows

Bij het debuggen van echte providers/modellen (vereist echte referenties):

- Live-suite (modellen + tool-/afbeeldingsprobes voor de Gateway): `pnpm test:live`
- Eén livebestand stil gericht uitvoeren: `pnpm test:live -- src/agents/models.profiles.live.test.ts`
- Rapporten over runtimeprestaties: dispatch `OpenClaw Performance` met
  `live_openai_candidate=true` voor een echte `openai/gpt-5.6-luna`-agentbeurt of
  `deep_profile=true` voor Kova-artefacten voor CPU/heap/tracering. Dagelijkse geplande uitvoeringen
  publiceren rapporten voor de mockprovider-, diep profiel- en GPT-5.6 Luna-lanes naar
  `openclaw/clawgrit-reports` vanuit een afzonderlijke publisher-taak die artefacten verwerkt;
  ontbrekende of ongeldige publisher-authenticatie laat geplande uitvoeringen en
  `profile=release`-uitvoeringen mislukken. Handmatige dispatches die geen release betreffen, behouden de GitHub-artefacten
  en behandelen rapportpublicatie als adviserend. Het mockproviderrapport bevat ook
  bronniveaugetallen voor het opstarten van de Gateway, geheugen, pluginbelasting, herhaalde
  hello-lussen van nepmodellen en het opstarten van de CLI.
- Docker-sweep van live modellen: `pnpm test:docker:live-models`
  - Elk geselecteerd model voert een tekstbeurt en een kleine probe in de stijl van het lezen van een bestand uit.
    Modellen waarvan de metagegevens `image`-invoer aangeven, voeren ook een kleine afbeeldingsbeurt uit.
    Schakel de extra probes uit met `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` of
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` wanneer je providerfouten isoleert.
  - CI-dekking: zowel de dagelijkse `OpenClaw Scheduled Live And E2E Checks` als de handmatige
    `OpenClaw Release Checks` roept de herbruikbare live/E2E-workflow aan met
    `include_live_suites: true`, die Docker-matrixtaken voor live modellen bevat,
    verdeeld per provider.
  - Dispatch voor gerichte CI-heruitvoeringen `OpenClaw Live And E2E Checks (Reusable)`
    met `include_live_suites: true` en `live_models_only: true`.
  - Voeg nieuwe providergeheimen met een sterk signaal toe aan `scripts/ci-hydrate-live-auth.sh`
    plus `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` en de
    geplande/release-aanroepers ervan.
- Rooktest voor native aan Codex gebonden chats: `pnpm test:docker:live-codex-bind`
  - Voert een Docker-live-lane uit via het app-serverpad van Codex, bindt een
    synthetisch Slack-DM met `/codex bind`, oefent `/codex fast` en
    `/codex permissions` uit en verifieert vervolgens dat een gewoon antwoord en een afbeeldingsbijlage
    via de native pluginbinding worden gerouteerd in plaats van via ACP.
- Rooktest van Codex-app-serverharnas: `pnpm test:docker:live-codex-harness`
  - Voert Gateway-agentbeurten uit via het app-serverharnas van Codex dat eigendom is
    van de plugin, verifieert `/codex status` en `/codex models` en oefent standaard
    afbeeldings-, cron-MCP-, subagent- en Guardian-probes uit. Schakel de
    subagentprobe uit met `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0` wanneer je
    andere fouten isoleert. Schakel voor een gerichte subagentcontrole de
    andere probes uit:
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`.
    Dit proces stopt na de subagentprobe, tenzij
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0` is ingesteld.
- Rooktest voor installatie van Codex op aanvraag: `pnpm test:docker:codex-on-demand`
  - Installeert het verpakte OpenClaw-tarballbestand in Docker, voert onboarding met
    een OpenAI-API-sleutel uit en verifieert dat de Codex-plugin plus de `@openai/codex`-afhankelijkheid
    op aanvraag naar de hoofdmap van het beheerde npm-project zijn gedownload.
- Live pakket-rooktest voor de Codex-npm-plugin: `pnpm test:docker:live-codex-npm-plugin`
  - Installeert het kandidaatpakket van OpenClaw en de exacte Codex-plugin in Docker
    en gebruikt vervolgens een echte OpenAI-sleutel voor CLI-preflight en beurten binnen dezelfde sessie.
  - De vervolgbeurt met nul nieuwe pogingen en gemiddeld denkniveau moet voortgang verzenden, blijven
    werken via willekeurige leesbewerkingen in de werkruimte en exact één artefact schrijven,
    en vervolgens voltooiing verzenden. Een terminale beurt met alleen voortgang laat de lane mislukken.
- Rooktest voor afhankelijkheden van live plugintools: `pnpm test:docker:live-plugin-tool`
  - Pakt een fixtureplugin met een echte `slugify`-afhankelijkheid in, installeert deze
    via `npm-pack:`, verifieert de afhankelijkheid onder de hoofdmap van het beheerde npm-project
    en vraagt vervolgens een live OpenAI-model om de plugintool aan te roepen en
    de verborgen slug terug te geven.
- Rooktest voor de OpenClaw-reddingsopdracht: `pnpm test:live:system-agent-rescue-channel`
  - Optionele extra zekerheidscontrole voor de interface van de reddingsopdracht
    voor berichtkanalen. Oefent `/openclaw status` uit, zet een permanente modelwijziging
    in de wachtrij, antwoordt `/openclaw yes` en verifieert het schrijfpad voor audit/configuratie.
- Docker-rooktest voor de eerste uitvoering van OpenClaw: `pnpm test:docker:system-agent-first-run`
  - Begint met een lege OpenClaw-statusmap en bewijst eerst dat de verpakte
    `openclaw setup`-CLI gesloten faalt zonder inferentie. Daarna
    test en activeert deze nep-Claude via de verpakte activeringsmodule.
    Pas daarna bereikt een onnauwkeurig verzoek aan de verpakte CLI de planner en
    wordt het omgezet in getypeerde installatie, gevolgd door eenmalige bewerkingen voor model,
    agent, Discord-configuratie en SecretRef. Het valideert configuratie- en auditvermeldingen. Dit is
    ondersteunend bewijs voor gates/bewerkingen, geen bewijs voor interactieve onboarding of
    OpenClaw-agenten/tools/goedkeuringen. Dezelfde lane is in QA Lab beschikbaar via
    `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup`.
- Moonshot/Kimi-kostenrooktest: voer met `MOONSHOT_API_KEY` ingesteld
  `openclaw models list --provider moonshot --json` uit en voer vervolgens een geïsoleerde
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json`
  uit tegen `moonshot/kimi-k2.6`. Verifieer dat de JSON Moonshot/K2.6 rapporteert en dat het
  assistenttranscript genormaliseerde `usage.cost` opslaat.

<Tip>
Wanneer je slechts één falend geval nodig hebt, kun je live tests het beste beperken via de hieronder beschreven allowlist-omgevingsvariabelen.
</Tip>

## QA-specifieke runners

Deze opdrachten staan naast de belangrijkste testsuites wanneer je het realisme van QA Lab nodig hebt.

CI voert QA Lab uit in specifieke workflows. Agentische pariteit valt onder
`QA-Lab - All Lanes` en releasevalidatie, niet onder een afzonderlijke PR-workflow.
Voor brede validatie moet `Full Release Validation` met
`rerun_group=qa-parity` of de QA-groep voor releasecontroles worden gebruikt. Stabiele/standaard
releasecontroles houden uitgebreide live-/Docker-duurtests achter `run_release_soak=true`; het
`full`-profiel dwingt duurtests af. `QA-Lab - All Lanes` wordt elke nacht uitgevoerd op `main` en
via handmatige dispatch met de mockpariteitslane, live Matrix-lane,
door Convex beheerde live Telegram-lane en door Convex beheerde live Discord-lane als
parallelle taken. Geplande QA- en releasecontroles voeren het Matrix-releaseprofiel
uit via de gedeelde liveadapter. De standaardwaarde van de Matrix-CLI en handmatige workflowinvoer
blijft `all`; handmatige `all`-dispatches splitsen zich uit over de transport-, media- en
E2EE-profielen, terwijl gerichte dispatches `fast`, `release` of
`transport` kunnen selecteren. `OpenClaw Release Checks` voert vóór releasegoedkeuring pariteit plus het
herbruikbare Matrix-liveadapterprofiel en de Telegram-lane uit. Transportcontroles voor releases
gebruiken `mock-openai/gpt-5.6-luna`, zodat ze deterministisch blijven en
het normale opstarten van providerplugins vermijden. Deze live transport-Gateways
schakelen geheugenzoekopdrachten uit; geheugengedrag blijft gedekt door de QA-pariteitssuites.

Volledige live mediashards voor releases gebruiken
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`, dat al
`ffmpeg` en `ffprobe` bevat. Docker-shards voor live modellen/backends gebruiken de gedeelde
`ghcr.io/openclaw/openclaw-live-test:<sha>`-image die eenmaal per geselecteerde
commit wordt gebouwd en halen deze vervolgens op met `OPENCLAW_SKIP_DOCKER_BUILD=1` in plaats van deze
binnen elke shard opnieuw te bouwen.

- `pnpm openclaw qa suite`
  - Voert QA-scenario's uit de repository rechtstreeks op de host uit.
  - Schrijft artefacten op het hoogste niveau naar `qa-evidence.json`, `qa-suite-summary.json` en
    `qa-suite-report.md` voor de geselecteerde scenarioset, inclusief
    selecties van gemengde flow-, Vitest- en Playwright-scenario's.
  - Bij aanroepen door `pnpm openclaw qa run --qa-profile <profile>` wordt
    de scorecard van het geselecteerde taxonomieprofiel in dezelfde `qa-evidence.json` ingesloten.
    `smoke-ci` schrijft beknopt bewijs (`evidenceMode: "slim"`, geen
    `execution` per item). `release` omvat de samengestelde selectie voor releasegereedheid; `all`
    selecteert elke actieve volwassenheidscategorie en richt zich op expliciete workflowaanroepen
    voor QA-profielbewijs wanneer een volledig scorecardartefact nodig is.
  - Voert meerdere geselecteerde scenario's standaard parallel uit met geïsoleerde
    Gateway-workers. `qa-channel` gebruikt standaard gelijktijdigheid 4 (begrensd door het
    aantal geselecteerde scenario's). Gebruik `--concurrency <count>` om het aantal
    workers aan te passen, of `--concurrency 1` voor de oudere seriële lane.
  - Eindigt met een niet-nulstatus wanneer een scenario mislukt. Gebruik `--allow-failures` voor
    artefacten zonder een mislukte afsluitcode.
  - Ondersteunt de providermodi `live-frontier`, `mock-openai` en `aimock`.
    `aimock` start een lokale, door AIMock ondersteunde providerserver voor experimentele
    fixture- en protocolmockdekking zonder de scenariobewuste
    `mock-openai`-lane te vervangen.
- `pnpm openclaw qa coverage --match <query>`
  - Doorzoekt scenario-ID's, titels, oppervlakken, dekkings-ID's, documentatiereferenties, codeverwijzingen,
    plugins en providervereisten en drukt vervolgens overeenkomende suitedoelen
    af.
  - Gebruik dit vóór een QA Lab-uitvoering wanneer het gewijzigde gedrag of bestandspad
    bekend is, maar niet het kleinste scenario. Alleen adviserend — kies nog steeds mock-,
    live-, Multipass-, Matrix- of transportbewijs op basis van het gedrag dat
    wordt gewijzigd.
- `pnpm test:plugins:kitchen-sink-live`
  - Voert de live OpenAI Kitchen Sink-pluginbeproeving uit via QA Lab.
    Installeert het externe Kitchen Sink-pakket, verifieert de inventaris van het
    plugin-SDK-oppervlak, test `/healthz` en `/readyz`, registreert bewijs van Gateway-
    CPU/RSS, voert een live OpenAI-beurt uit en controleert vijandige
    diagnostiek. Vereist live OpenAI-authenticatie, zoals `OPENAI_API_KEY`. In
    gehydrateerde Testbox-sessies wordt automatisch het live-authenticatieprofiel van Testbox
    geladen wanneer de helper `openclaw-testbox-env` aanwezig is.
- `pnpm test:gateway:cpu-scenarios`
  - Voert de Gateway-opstartbenchmark plus een klein pakket mockscenario's van QA Lab uit
    (`channel-chat-baseline`, `memory-failure-fallback`,
    `gateway-restart-inflight-run`) en schrijft een gecombineerd overzicht
    van CPU-waarnemingen onder `.artifacts/gateway-cpu-scenarios/`.
  - Markeert standaard alleen aanhoudende waarnemingen van hoge CPU-belasting (`--cpu-core-warn`,
    standaard `0.9`; `--hot-wall-warn-ms`, standaard `30000`), zodat korte
    opstartpieken als metrieken worden geregistreerd zonder te lijken op de regressie
    waarbij de Gateway minutenlang de CPU volledig belast.
  - Wordt uitgevoerd met gebouwde `dist`-artefacten; voer eerst een build uit wanneer de checkout
    nog geen recente runtime-uitvoer bevat.
- `pnpm openclaw qa suite --runner multipass`
  - Voert dezelfde QA-suite uit in een wegwerpbare Multipass Linux-VM, met
    dezelfde flags voor scenarioselectie en provider/model als `qa suite`.
  - Live-uitvoeringen sturen de voor de guest bruikbare QA-authenticatie-invoer door:
    providerkeys uit omgevingsvariabelen, het configuratiepad van de live QA-provider en
    `CODEX_HOME` indien aanwezig.
  - Uitvoermappen moeten onder de hoofdmap van de repository blijven, zodat de guest via
    de gekoppelde werkruimte kan terugschrijven.
  - Schrijft het normale QA-rapport en -overzicht plus Multipass-logboeken onder
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Start de door Docker ondersteunde QA-site voor QA-werk in operatorstijl.
- `pnpm test:docker:npm-onboard-channel-agent`
  - Bouwt een npm-tarball vanuit de huidige checkout, installeert deze globaal in
    Docker, voert niet-interactieve onboarding met een OpenAI-API-key uit, configureert
    standaard Telegram, verifieert dat de verpakte pluginruntime wordt geladen zonder
    reparatie van opstartafhankelijkheden, voert doctor uit en voert één lokale agentbeurt
    uit tegen een gemockt OpenAI-eindpunt.
  - Gebruik `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` om dezelfde lane voor
    verpakte installatie uit te voeren met Discord.
- `pnpm test:docker:session-runtime-context`
  - Voert een deterministische Docker-smoketest van de gebouwde app uit voor transcripten met
    ingesloten runtimecontext. Verifieert dat verborgen OpenClaw-runtimecontext behouden blijft als een
    niet-weergegeven aangepast bericht in plaats van in de zichtbare gebruikersbeurt te lekken,
    initialiseert vervolgens een betrokken, defecte JSONL-sessie en verifieert dat
    `openclaw doctor --fix` deze met een back-up naar de actieve branch herschrijft.
- `pnpm test:docker:npm-telegram-live`
  - Installeert een kandidaat-OpenClaw-pakket in Docker, voert onboarding van het
    geïnstalleerde pakket uit, configureert Telegram via de geïnstalleerde CLI en hergebruikt
    vervolgens de live QA-lane voor Telegram met dat geïnstalleerde pakket als de
    Gateway van het systeem onder test.
  - De wrapper koppelt alleen de broncode van de `qa-lab`-testinfrastructuur vanuit de checkout;
    het geïnstalleerde pakket beheert `dist`, `openclaw/plugin-sdk` en de runtime van
    gebundelde plugins, zodat de lane geen plugins uit de huidige checkout mengt met
    het geteste pakket.
  - Gebruikt standaard `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta`; stel
    `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` of
    `OPENCLAW_CURRENT_PACKAGE_TGZ` in om een opgeloste lokale tarball te testen in plaats
    van uit het register te installeren.
  - Produceert standaard herhaalde RTT-timing in `qa-evidence.json` met
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20`. Overschrijf
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`,
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS` of
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` om de uitvoering aan te passen.
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS` selecteert het QA-scenario voor Telegram
    dat wordt bemonsterd; het ondersteunde RTT-doel is `channel-canary`.
  - Gebruikt dezelfde Telegram-omgevingsreferenties of Convex-referentiebron als
    `pnpm openclaw qa telegram`. Stel voor CI/releaseautomatisering
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex` plus
    `OPENCLAW_QA_CONVEX_SITE_URL` en een rolgeheim in. Als
    `OPENCLAW_QA_CONVEX_SITE_URL` en een Convex-rolgeheim aanwezig zijn in
    CI, selecteert de Docker-wrapper automatisch Convex.
  - De wrapper valideert de omgevingsvariabelen voor Telegram- of Convex-referenties op de host
    voordat Docker-build- en installatiewerk begint. Stel
    `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1` alleen in wanneer
    je bewust de configuratie vóór de referenties debugt.
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer` overschrijft
    de gedeelde `OPENCLAW_QA_CREDENTIAL_ROLE` uitsluitend voor deze lane. Wanneer Convex-
    referenties zijn geselecteerd en geen rol is ingesteld, gebruikt de wrapper `ci` in CI
    en `maintainer` buiten CI.
  - GitHub Actions stelt deze lane beschikbaar als de handmatige maintainerworkflow
    `NPM Telegram Beta E2E`. Deze wordt niet uitgevoerd bij samenvoegen. De workflow gebruikt de
    `qa-live-shared`-omgeving en Convex CI-referentieleases.
- GitHub Actions stelt ook `Package Acceptance` beschikbaar voor aanvullend productbewijs
  tegen één kandidaatpakket. Het accepteert een Git-referentie, gepubliceerde npm-specificatie,
  HTTPS-tarball-URL plus SHA-256, beleid voor vertrouwde URL's of een tarballartefact
  uit een andere uitvoering (`source=ref|npm|url|trusted-url|artifact`), uploadt de
  genormaliseerde `openclaw-current.tgz` als `package-under-test` en voert vervolgens de
  bestaande Docker E2E-planner uit met laneprofielen `smoke`, `package`, `product`, `full`
  of `custom`. Stel `telegram_mode=mock-openai` of
  `live-frontier` in om de QA-workflow voor Telegram uit te voeren tegen hetzelfde
  `package-under-test`-artefact.
  - Productbewijs voor de nieuwste bèta:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- Bewijs met een exacte tarball-URL vereist een digest en gebruikt het veiligheidsbeleid voor openbare URL's:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- Zakelijke/private tarballmirrors gebruiken een expliciet beleid voor vertrouwde bronnen:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url` leest `.github/package-trusted-sources.json` uit de vertrouwde workflowreferentie en accepteert geen URL-referenties of een via workflowinvoer ingestelde omzeiling van het privénetwerk. Als het benoemde beleid bearer-authenticatie declareert, configureer je het vaste geheim `OPENCLAW_TRUSTED_PACKAGE_TOKEN`.

- Artefactbewijs downloadt een tarballartefact uit een andere Actions-uitvoering:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - Verpakt en installeert de huidige OpenClaw-build in Docker, start de
    Gateway met geconfigureerde OpenAI en schakelt vervolgens gebundelde kanalen/plugins in via
    configuratiewijzigingen.
  - Verifieert dat setupdetectie niet-geconfigureerde downloadbare plugins
    afwezig laat, dat de eerste geconfigureerde doctorreparatie elke ontbrekende
    downloadbare plugin expliciet installeert en dat een tweede herstart geen
    verborgen afhankelijkheidsreparatie uitvoert.
  - Installeert ook een bekende oudere npm-basisversie, schakelt Telegram in voordat
    `openclaw update --tag <candidate>` wordt uitgevoerd en verifieert dat
    doctor van de kandidaat na de update verouderde resten van pluginafhankelijkheden opruimt
    zonder een postinstall-reparatie vanuit de testinfrastructuur.
- `pnpm test:parallels:npm-update`
  - Voert de native smoketest voor updates van verpakte installaties uit op Parallels-guests.
    Elk geselecteerd platform installeert eerst het gevraagde basispakket,
    voert vervolgens de geïnstalleerde opdracht `openclaw update` uit in dezelfde guest en
    verifieert de geïnstalleerde versie, updatestatus, gereedheid van de Gateway en
    één lokale agentbeurt.
  - Gebruik `--platform macos`, `--platform windows` of `--platform linux`
    tijdens iteratie op één guest. Gebruik `--json` voor het pad van het overzichtsartefact
    en de status per lane.
  - De OpenAI-lane gebruikt standaard `openai/gpt-5.6-luna` voor het livebewijs van de agentbeurt.
    Geef `--model <provider/model>` door of stel
    `OPENCLAW_PARALLELS_OPENAI_MODEL` in om een ander OpenAI-model te valideren.
  - Plaats langdurige lokale uitvoeringen binnen een hosttime-out, zodat vastgelopen Parallels-transport
    niet de rest van het testvenster kan verbruiken:

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - Het script schrijft geneste lanelogboeken onder
    `/tmp/openclaw-parallels-npm-update.*`. Controleer `windows-update.log`,
    `macos-update.log` of `linux-update.log` voordat je aanneemt dat de buitenste
    wrapper is vastgelopen.
  - De Windows-update kan op een koude guest 10 tot 15 minuten besteden aan doctor na de update en
    pakketupdates; dit is nog steeds normaal wanneer het geneste npm-debuglogboek
    voortgang vertoont.
  - Voer deze geaggregeerde wrapper niet parallel uit met afzonderlijke Parallels-
    smoketestlanes voor macOS, Windows of Linux. Ze delen VM-status en kunnen
    botsen bij het herstellen van snapshots, aanbieden van pakketten of de Gateway-status van de guest.
  - Het bewijs na de update voert het normale oppervlak van gebundelde plugins uit, omdat
    capaciteitsfacades zoals spraak, afbeeldingsgeneratie en mediabegrip
    via gebundelde runtime-API's worden geladen, zelfs wanneer de agentbeurt
    zelf alleen een eenvoudig tekstantwoord controleert.

- `pnpm openclaw qa aimock`
  - Start alleen de lokale AIMock-providerserver voor directe protocolrooktests.
- `pnpm openclaw qa matrix`
  - Voert de live QA-lane voor Matrix uit tegen een tijdelijke, door Docker ondersteunde Tuwunel-homeserver. Alleen voor broncodecheck-outs - verpakte installaties leveren
    `qa-lab` niet mee.
  - Volledige CLI, catalogus met profielen/scenario's, omgevingsvariabelen en artefactindeling:
    [Matrix-rooktestlanes](/nl/concepts/qa-e2e-automation#matrix-smoke-lanes).
- `pnpm openclaw qa telegram`
  - Voert de live QA-lane voor Telegram uit tegen een echte privégroep met de
    tokens voor de driver- en SUT-bot uit de omgeving.
  - Vereist `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` en
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. De groeps-id moet de numerieke
    Telegram-chat-id zijn.
  - Ondersteunt `--credential-source convex` voor gedeelde, gepoolde referenties.
    Gebruik standaard de omgevingsmodus of stel `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` in
    om gepoolde leases te gebruiken.
  - De standaardinstellingen dekken canary, vermeldingstoegang, commandoadressering, `/status`,
    vermelde antwoorden van bot naar bot en antwoorden op native kerncommando's.
    De standaardinstellingen van `mock-openai` dekken ook deterministische antwoordketens en
    regressies in het streamen van definitieve Telegram-berichten. Gebruik `--list-scenarios`
    voor optionele controles zoals `session_status`.
  - Sluit af met een niet-nulstatus wanneer een scenario mislukt. Gebruik `--allow-failures` voor
    artefacten zonder een falende afsluitcode.
  - Vereist twee verschillende bots in dezelfde privégroep, waarbij de SUT-bot
    een Telegram-gebruikersnaam beschikbaar stelt.
  - Schakel voor stabiele observatie tussen bots Bot-to-Bot Communication Mode
    in `@BotFather` in voor beide bots en zorg dat de driverbot
    botverkeer in de groep kan waarnemen.
  - Schrijft een Telegram-QA-rapport, samenvatting en `qa-evidence.json` naar
    `.artifacts/qa-e2e/...`. Antwoordscenario's bevatten de RTT vanaf het
    verzendverzoek van de driver tot het waargenomen antwoord van de SUT.

`Mantis Telegram Live` is de wrapper voor PR-bewijs rond deze lane. Deze voert
de kandidaatref uit met via Convex geleasete Telegram-referenties, geeft de
geredigeerde QA-rapport-/bewijsbundel weer in een Crabbox-desktopbrowser, neemt
MP4-bewijs op, genereert een op beweging bijgesneden GIF, uploadt de artefactbundel en
plaatst inline PR-bewijs via de Mantis GitHub App wanneer `pr_number` is
ingesteld. Onderhouders kunnen deze starten vanuit de Actions-interface via `Mantis Scenario`
(`scenario_id: telegram-live`) of rechtstreeks vanuit een opmerking bij een pull request:

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof` is de agentische native wrapper voor
Telegram Desktop vóór/na visueel PR-bewijs. Start deze vanuit de Actions-interface met
vrije invoer voor `instructions`, via `Mantis Scenario` (`scenario_id:
telegram-desktop-proof`) of vanuit een PR-opmerking:

```text
@openclaw-mantis telegram desktop proof
```

De Mantis-agent leest de PR, bepaalt welk voor Telegram zichtbaar gedrag
de wijziging bewijst, voert de echte-gebruikersbewijs-lane voor Crabbox Telegram Desktop uit op
basis- en kandidaatreferenties, herhaalt dit totdat de native GIF's bruikbaar zijn,
schrijft een gekoppeld `motionPreview`-manifest en plaatst dezelfde GIF-tabel met 2 kolommen
via de Mantis GitHub App wanneer `pr_number` is ingesteld.

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - Leaset of hergebruikt een Crabbox Linux-desktop, installeert native Telegram
    Desktop, configureert OpenClaw met een geleaset Telegram-SUT-bottoken,
    start de Gateway en neemt schermafbeeldings-/MP4-bewijs op vanaf de
    zichtbare VNC-desktop.
  - Gebruikt standaard `--credential-source convex`, zodat workflows alleen het
    geheim van de Convex-broker nodig hebben. Gebruik `--credential-source env` met dezelfde
    `OPENCLAW_QA_TELEGRAM_*`-variabelen als `pnpm openclaw qa telegram`.
  - Telegram Desktop vereist nog steeds een gebruikersaanmelding/-profiel. Het bottoken
    configureert alleen OpenClaw. Gebruik `--telegram-profile-archive-env <name>`
    voor een base64-`.tgz`-profielarchief, of gebruik `--keep-lease` en meld je
    eenmaal handmatig aan via VNC.
  - Schrijft `mantis-telegram-desktop-builder-report.md`,
    `mantis-telegram-desktop-builder-summary.json`,
    `telegram-desktop-builder.png` en `telegram-desktop-builder.mp4`
    naar de uitvoermap.

Live transportlanes delen één standaardcontract, zodat nieuwe transporten niet
uiteenlopen; de dekkingsmatrix per lane staat in
[QA-overzicht - Live transportdekking](/nl/concepts/qa-e2e-automation#live-transport-coverage).
`qa-channel` is de brede synthetische suite en maakt geen deel uit van die matrix.

### Gedeelde Telegram-referenties via Convex (v1)

Wanneer `--credential-source convex` (of `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`)
is ingeschakeld voor live transport-QA, verkrijgt het QA-lab een exclusieve lease uit een
door Convex ondersteunde pool, stuurt het Heartbeats voor die lease zolang de lane actief is en
geeft het de lease bij afsluiten vrij. De sectienaam dateert van vóór ondersteuning voor Discord, Slack en
WhatsApp; het leasecontract wordt door alle typen gedeeld.

Referentiesteiger voor het Convex-project: `qa/convex-credential-broker/`

Vereiste omgevingsvariabelen:

- `OPENCLAW_QA_CONVEX_SITE_URL` (bijvoorbeeld `https://your-deployment.convex.site`)
- Eén geheim voor de geselecteerde rol:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` voor `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` voor `ci`
- Selectie van referentierol:
  - CLI: `--credential-role maintainer|ci`
  - Standaardwaarde uit omgeving: `OPENCLAW_QA_CREDENTIAL_ROLE` (standaard `ci` in CI, anders `maintainer`)

Optionele omgevingsvariabelen:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (standaard `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (standaard `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (standaard `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (standaard `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (standaard `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (optionele traceer-id)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` staat loopback-`http://`-URL's van Convex toe voor uitsluitend lokale ontwikkeling.

`OPENCLAW_QA_CONVEX_SITE_URL` moet bij normaal gebruik `https://` gebruiken.

Beheerderscommando's voor onderhouders (pool toevoegen/verwijderen/tonen) vereisen
specifiek `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`.

CLI-hulpmiddelen voor onderhouders:

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Gebruik `doctor` vóór live uitvoeringen om de Convex-site-URL, brokergeheimen,
endpointprefix, HTTP-time-out en bereikbaarheid van beheer/lijsten te controleren zonder
geheime waarden af te drukken. Gebruik `--json` voor machineleesbare uitvoer in scripts en
CI-hulpprogramma's.

Standaard endpointcontract (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`).
Verzoeken worden geverifieerd met een `Authorization: Bearer <role secret>`-header;
in de onderstaande bodies is die header weggelaten:

- `POST /acquire`
  - Verzoek: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Geslaagd: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Uitgeput/opnieuw te proberen: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - Verzoek: `{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - Geslaagd: `{ status: "ok", index, data }`
- `POST /heartbeat`
  - Verzoek: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Geslaagd: `{ status: "ok" }` (of lege `2xx`)
- `POST /release`
  - Verzoek: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Geslaagd: `{ status: "ok" }` (of lege `2xx`)
- `POST /admin/add` (alleen onderhoudersgeheim)
  - Verzoek: `{ kind, actorId, payload, note?, status? }`
  - Geslaagd: `{ status: "ok", credential }`
- `POST /admin/remove` (alleen onderhoudersgeheim)
  - Verzoek: `{ credentialId, actorId }`
  - Geslaagd: `{ status: "ok", changed, credential }`
  - Beveiliging voor actieve lease: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (alleen onderhoudersgeheim)
  - Verzoek: `{ kind?, status?, includePayload?, limit? }`
  - Geslaagd: `{ status: "ok", credentials, count }`

Payloadvorm voor het Telegram-type:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` moet een numerieke Telegram-chat-id-tekenreeks zijn.
- `admin/add` valideert deze vorm voor `kind: "telegram"` en weigert onjuist gevormde payloads.

Payloadvorm voor het echte-gebruikerstype van Telegram:

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`, `testerUserId` en `telegramApiId` moeten numerieke tekenreeksen zijn.
- `tdlibArchiveSha256` en `desktopTdataArchiveSha256` moeten hexadecimale SHA-256-tekenreeksen zijn.
- `kind: "telegram-user"` is gereserveerd voor de bewijsworkflow voor Mantis Telegram Desktop. Generieke QA-lablanes mogen deze niet verkrijgen.

Door de broker gevalideerde payloads voor meerdere kanalen:

- Discord: `{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp: `{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

Slack-lanes kunnen ook uit de pool leasen, maar de validatie van Slack-payloads
bevindt zich momenteel in de Slack-QA-runner in plaats van in de broker. Gebruik
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`
voor Slack-rijen.

### Een kanaal aan QA toevoegen

De architectuur en namen van scenariohulpmiddelen voor nieuwe kanaaladapters staan in
[QA-overzicht - Een kanaal toevoegen](/nl/concepts/qa-e2e-automation#adding-a-channel).
De minimumeis: implementeer de transportrunner op de gedeelde `qa-lab`-hostnaad,
voeg een `adapterFactory` toe voor gedeelde scenario's, declareer `qaRunners` in het
Pluginmanifest, koppel als `openclaw qa <runner>` en schrijf scenario's onder
`qa/scenarios/`.

## Testsuites (wat waar wordt uitgevoerd)

Beschouw de suites als een reeks met "toenemend realisme" (en toenemende instabiliteit/kosten).

### Unit-/integratietests (standaard)

- Commando: `pnpm test`
- Configuratie: niet-gerichte uitvoeringen gebruiken de `vitest.full-*.config.ts`-shardset en kunnen
  shards voor meerdere projecten uitbreiden naar configuraties per project voor parallelle
  planning
- Bestanden: kern-/unitinventarissen onder `src/**/*.test.ts`,
  `packages/**/*.test.ts` en `test/**/*.test.ts`; UI-unittests worden uitgevoerd in de
  speciale `unit-ui`-shard
- Bereik:
  - Zuivere unittests
  - Integratietests binnen het proces (Gateway-authenticatie, routering, hulpmiddelen, parsering, configuratie)
  - Deterministische regressietests voor bekende bugs
- Verwachtingen:
  - Wordt uitgevoerd in CI
  - Geen echte sleutels vereist
  - Moet snel en stabiel zijn
  - Tests voor resolvers en loaders van publieke oppervlakken moeten breed `api.js`- en
    `runtime-api.js`-fallbackgedrag aantonen met gegenereerde, kleine Pluginfixtures,
    niet met echte bron-API's van meegeleverde Plugins. Echte Plugin-API-ladingen horen thuis in
    contract-/integratiesuites die eigendom zijn van de Plugin.

Beleid voor native afhankelijkheden:

- Standaard testinstallaties slaan optionele native Discord-opusbuilds over. Discord-
  spraak gebruikt de meegeleverde `libopus-wasm` en `@discordjs/opus` blijft uitgeschakeld in
  `allowBuilds`, zodat lokale tests en Testbox-lanes de native
  add-on niet compileren.
- Vergelijk native opusprestaties in de benchmarkrepository `libopus-wasm`, niet
  in standaard installatie-/testlussen van OpenClaw. Stel `@discordjs/opus` niet in op
  `true` in de standaard-`allowBuilds`; daardoor compileren niet-gerelateerde installatie-/testlussen
  native code.

<AccordionGroup>
  <Accordion title="Projecten, shards en afgebakende lanes">

    - Niet-gerichte `pnpm test` voert dertien kleinere shardconfiguraties uit (`core-unit-fast`, `core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-tooling`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) in plaats van één enorm native hoofdprojectproces. Dit verlaagt de RSS-piek op belaste machines en voorkomt dat automatisch antwoord-/pluginwerk niet-gerelateerde suites uithongert.
    - `pnpm test --watch` gebruikt nog steeds de native projectgraaf van het hoofdproject `vitest.config.ts`, omdat een watch-lus met meerdere shards niet praktisch is.
    - `pnpm test`, `pnpm test:watch` en `pnpm test:perf:imports` leiden expliciete bestands-/mapdoelen eerst door bereikgebonden lanes, zodat `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` niet de volledige opstartkosten van het hoofdproject draagt.
    - `pnpm test:changed` breidt gewijzigde git-paden standaard uit naar goedkope bereikgebonden lanes: rechtstreekse testbewerkingen, naastgelegen `*.test.ts`-bestanden, expliciete brontoewijzingen en afhankelijke onderdelen uit de lokale importgraaf. Bewerkingen aan configuratie, installatie of pakketten voeren geen brede tests uit, tenzij je expliciet `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` gebruikt.
    - `pnpm check:changed` is de gebruikelijke slimme lokale controlepoort voor beperkt werk. Deze classificeert de diff in kerncode, kerntests, extensies, extensietests, apps, documentatie, releasemetagegevens, live Docker-tooling en tooling, en voert vervolgens de bijpassende typecontrole-, lint- en beveiligingsopdrachten uit. Vitest-tests worden niet uitgevoerd; roep `pnpm test:changed` of expliciet `pnpm test <target>` aan voor testbewijs. Versieverhogingen met alleen releasemetagegevens voeren gerichte controles voor versies, configuratie en hoofdafhankelijkheden uit, met een beveiliging die pakketwijzigingen buiten het versieveld op het hoogste niveau afwijst.
    - Bewerkingen aan de live Docker ACP-harness voeren gerichte controles uit: shellsyntaxis voor de live Docker-authenticatiescripts en een dry-run van de live Docker-planner. Wijzigingen aan `package.json` worden alleen meegenomen wanneer de diff beperkt is tot `scripts["test:docker:live-*"]`; wijzigingen aan afhankelijkheden, exports, versies en andere pakketoppervlakken gebruiken nog steeds de bredere beveiligingen.
    - Importlichte unittests uit agents, opdrachten, plugins, hulpmiddelen voor automatisch antwoorden, `plugin-sdk` en vergelijkbare gebieden met pure hulpprogramma's worden via de `unit-fast`-lane geleid, die `test/setup-openclaw-runtime.ts` overslaat; bestanden met veel status- of runtimegebruik blijven op de bestaande lanes.
    - Geselecteerde bronbestanden van `plugin-sdk`- en `commands`-hulpprogramma's wijzen uitvoeringen in gewijzigde modus ook toe aan expliciete naastgelegen tests in die lichte lanes, zodat bewerkingen aan hulpprogramma's niet opnieuw de volledige zware suite voor die map uitvoeren.
    - `auto-reply` heeft afzonderlijke groepen voor kernhulpprogramma's op het hoogste niveau, `reply.*`-integratietests op het hoogste niveau en de `src/auto-reply/reply/**`-subboom. CI splitst de antwoordsubboom verder op in shards voor de agent-runner, dispatch en opdracht-/statusroutering, zodat één importzware groep niet de volledige Node-staart voor zijn rekening neemt.
    - Normale CI voor PR's/main slaat bewust de batchsweep van gebundelde plugins en de uitsluitend voor releases bestemde `agentic-plugins`-shard over. Volledige releasevalidatie activeert de afzonderlijke onderliggende `Plugin Prerelease`-workflow voor deze pluginzware suites op releasekandidaten.

  </Accordion>

  <Accordion title="Dekking van de ingesloten runner">

    - Wanneer je invoer voor detectie van berichttools of de runtimecontext voor Compaction wijzigt, moet je beide dekkingsniveaus behouden.
    - Voeg gerichte regressietests voor hulpprogramma's toe voor grenzen van pure routering en normalisatie.
    - Houd de integratiesuites voor de ingesloten runner gezond:
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`,
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts` en
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`.
    - Deze suites verifiëren dat bereikgebonden id's en Compaction-gedrag nog steeds door de echte `run.ts`- / `compact.ts`-paden stromen; tests die alleen hulpprogramma's testen, zijn geen afdoende vervanging voor deze integratiepaden.

  </Accordion>

  <Accordion title="Standaardwaarden voor Vitest-pools en -isolatie">

    - De basisconfiguratie van Vitest gebruikt standaard `threads`.
    - De gedeelde Vitest-configuratie legt `isolate: false` vast en gebruikt de niet-geïsoleerde runner voor de hoofdprojecten en de e2e- en live-configuraties.
    - De UI-lane van het hoofdproject behoudt de `jsdom`-installatie en -optimalisatie, maar draait ook op de gedeelde niet-geïsoleerde runner.
    - Elke `pnpm test`-shard neemt dezelfde standaardwaarden voor `threads` + `isolate: false` over uit de gedeelde Vitest-configuratie.
    - `scripts/run-vitest.mjs` voegt standaard `--no-maglev` toe voor onderliggende Node-processen van Vitest om V8-compilatieoverhead tijdens grote lokale uitvoeringen te verminderen.
      Stel `OPENCLAW_VITEST_ENABLE_MAGLEV=1` in om dit met het standaardgedrag van V8 te vergelijken.
    - `scripts/run-vitest.mjs` beëindigt expliciete Vitest-uitvoeringen buiten de watch-modus na 5 minuten zonder uitvoer naar stdout of stderr. Stel `OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0` in om de watchdog uit te schakelen voor een onderzoek dat opzettelijk geen uitvoer produceert.

  </Accordion>

  <Accordion title="Snelle lokale iteratie">

    - `pnpm changed:lanes` toont welke architectuurlanes door een diff worden geactiveerd.
    - De pre-commithaak voert alleen formattering uit. Deze voegt geformatteerde bestanden opnieuw toe aan de staging area en voert geen lint, typecontrole of tests uit.
    - Voer `pnpm check:changed` expliciet uit vóór overdracht of push wanneer je de slimme lokale controlepoort nodig hebt.
    - `pnpm test:changed` wordt standaard via goedkope bereikgebonden lanes geleid. Gebruik `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` alleen wanneer de agent bepaalt dat een bewerking aan een harness, configuratie, pakket of contract werkelijk bredere Vitest-dekking nodig heeft.
    - `pnpm test:max` en `pnpm test:changed:max` behouden hetzelfde routeringsgedrag, maar met een hogere limiet voor workers.
    - Automatisch schalen van lokale workers is bewust conservatief en schaalt terug wanneer de gemiddelde hostbelasting al hoog is, zodat meerdere gelijktijdige Vitest-uitvoeringen standaard minder schade aanrichten.
    - De basisconfiguratie van Vitest markeert de projecten/configuratiebestanden als `forceRerunTriggers`, zodat herhaalde uitvoeringen in gewijzigde modus correct blijven wanneer de testbedrading verandert.
    - De configuratie houdt `OPENCLAW_VITEST_FS_MODULE_CACHE` ingeschakeld op ondersteunde hosts; stel `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` in voor één expliciete cachelocatie voor rechtstreekse profilering.

  </Accordion>

  <Accordion title="Prestatieproblemen opsporen">

    - `pnpm test:perf:imports` schakelt rapportage van Vitest-importduur en uitvoer met importuitsplitsingen in.
    - `pnpm test:perf:imports:changed` beperkt dezelfde profileringsweergave tot bestanden die sinds `origin/main` zijn gewijzigd.
    - Timinggegevens van shards worden naar `.artifacts/vitest-shard-timings.json` geschreven.
      Uitvoeringen van volledige configuraties gebruiken het configuratiepad als sleutel; CI-shards met inclusiepatronen voegen de shardnaam toe, zodat gefilterde shards afzonderlijk kunnen worden gevolgd.
    - Wanneer één intensieve test nog steeds de meeste tijd besteedt aan imports tijdens het opstarten, houd je zware afhankelijkheden achter een smalle lokale `*.runtime.ts`-grens en mock je die grens rechtstreeks, in plaats van runtimehulpprogramma's diep te importeren om ze alleen via `vi.mock(...)` door te geven.
    - `pnpm test:perf:changed:bench -- --ref <git-ref>` vergelijkt gerouteerde `test:changed` met het native hoofdprojectpad voor die vastgelegde diff en toont de verstreken tijd plus het maximale RSS op macOS.
    - `pnpm test:perf:changed:bench -- --worktree` benchmarkt de huidige vuile werkboom door de lijst met gewijzigde bestanden via `scripts/test-projects.mjs` en de hoofdconfiguratie van Vitest te leiden.
    - `pnpm test:perf:profile:main` schrijft een CPU-profiel van de hoofdthread voor de opstart- en transformatieoverhead van Vitest/Vite.
    - `pnpm test:perf:profile:runner` schrijft CPU- en heap-profielen van de runner voor de unitsuite, waarbij bestandsparallellisme is uitgeschakeld.

  </Accordion>
</AccordionGroup>

### Stabiliteit (Gateway)

- Opdracht: `pnpm test:stability:gateway`
- Configuratie: `test/vitest/vitest.gateway.config.ts`, `test/vitest/vitest.logging.config.ts` en `test/vitest/vitest.infra.config.ts`, elk gedwongen tot één worker
- Bereik:
  - Start standaard een echte loopback-Gateway met diagnostiek ingeschakeld
  - Leidt synthetisch verloop van Gateway-berichten, geheugen en grote payloads door het diagnostische gebeurtenispad
  - Vraagt `diagnostics.stability` op via de Gateway WS-RPC
  - Dekt persistentiehulpprogramma's voor de diagnostische stabiliteitsbundel
  - Controleert dat de recorder begrensd blijft, synthetische RSS-monsters onder het drukbudget blijven en wachtrijdiepten per sessie weer tot nul leeglopen
- Verwachtingen:
  - Veilig voor CI en zonder sleutels
  - Smalle lane voor opvolging van stabiliteitsregressies, geen vervanging voor de volledige Gateway-suite

### E2E (repo-aggregaat)

- Opdracht: `pnpm test:e2e`
- Bereik:
  - Voert de E2E-lane voor de Gateway-smoketest uit
  - Voert de E2E-lane voor de gemockte Control UI-browser uit
- Verwachtingen:
  - Veilig voor CI en zonder sleutels
  - Vereist dat Playwright Chromium is geïnstalleerd

### E2E (Gateway-smoketest)

- Opdracht: `pnpm test:e2e:gateway`
- Configuratie: `test/vitest/vitest.e2e.config.ts`
- Bestanden: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts` en E2E-tests voor gebundelde plugins onder `extensions/`
- Standaardwaarden voor runtime:
  - Gebruikt Vitest `threads` met `isolate: false`, in overeenstemming met de rest van de repo.
  - Gebruikt adaptieve workers (CI: maximaal 2, lokaal: standaard 1).
  - Draait standaard in stille modus om overhead door console-I/O te verminderen.
- Nuttige overschrijvingen:
  - `OPENCLAW_E2E_WORKERS=<n>` om het aantal workers af te dwingen (begrensd op 16).
  - `OPENCLAW_E2E_VERBOSE=1` om uitgebreide console-uitvoer weer in te schakelen.
- Bereik:
  - End-to-endgedrag van de Gateway met meerdere instanties
  - WebSocket-/HTTP-oppervlakken, koppeling van nodes en zwaarder netwerkgebruik
- Verwachtingen:
  - Draait in CI (wanneer ingeschakeld in de pijplijn)
  - Geen echte sleutels vereist
  - Meer bewegende onderdelen dan unittests (kan langzamer zijn)

### E2E (gemockte Control UI-browser)

- Opdracht: `pnpm test:ui:e2e`
- Configuratie: `test/vitest/vitest.ui-e2e.config.ts`
- Bestanden: `ui/src/**/*.e2e.test.ts`
- Bereik:
  - Start de Vite Control UI
  - Stuurt via Playwright een echte Chromium-pagina aan
  - Vervangt de Gateway-WebSocket door deterministische mocks in de browser
- Verwachtingen:
  - Draait in CI als onderdeel van `pnpm test:e2e`
  - Geen echte Gateway, agents of providersleutels vereist
  - Browserafhankelijkheid moet aanwezig zijn (`pnpm --dir ui exec playwright install chromium`)

### E2E: OpenShell-backendsmoketest

- Opdracht: `pnpm test:e2e:openshell`
- Bestand: `extensions/openshell/src/backend.e2e.test.ts`
- Bereik:
  - Hergebruikt een actieve lokale OpenShell-Gateway
  - Maakt een sandbox vanuit een tijdelijk lokaal Dockerfile
  - Test de OpenShell-backend van OpenClaw via echte `sandbox ssh-config` + SSH-uitvoering
  - Verifieert op afstand canoniek bestandssysteemgedrag via de fs-bridge van de sandbox
- Verwachtingen:
  - Alleen opt-in; maakt geen deel uit van de standaarduitvoering van `pnpm test:e2e`
  - Vereist een lokale `openshell`-CLI plus een werkende Docker-daemon
  - Vereist een actieve lokale OpenShell-Gateway en de bijbehorende configuratiebron
  - Gebruikt geïsoleerde `HOME` / `XDG_CONFIG_HOME` en vernietigt daarna de testsandbox
- Nuttige overschrijvingen:
  - `OPENCLAW_E2E_OPENSHELL=1` om de test in te schakelen wanneer je de bredere e2e-suite handmatig uitvoert
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` om naar een niet-standaard CLI-binair bestand of wrapperscript te verwijzen
  - `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config` om de geregistreerde Gateway-configuratie beschikbaar te maken voor de geïsoleerde test
  - `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1` om het Docker-Gateway-IP te overschrijven dat door de hostbeleidsfixture wordt gebruikt

### Live (echte providers + echte modellen)

- Opdracht: `pnpm test:live`
- Configuratie: `test/vitest/vitest.live.config.ts`
- Bestanden: `src/**/*.live.test.ts`, `test/**/*.live.test.ts` en live tests voor gebundelde plugins onder `extensions/`
- Standaard: **ingeschakeld** door `pnpm test:live` (stelt `OPENCLAW_LIVE_TEST=1` in)
- Bereik:
  - "Werkt deze provider/dit model _vandaag_ daadwerkelijk met echte inloggegevens?"
  - Signaleer wijzigingen in providerindelingen, eigenaardigheden bij het aanroepen van tools, authenticatieproblemen en gedrag bij frequentielimieten
- Verwachtingen:
  - Opzettelijk niet CI-stabiel (echte netwerken, echt providerbeleid, quota, storingen)
  - Kost geld / gebruikt frequentielimieten
  - Voer bij voorkeur beperkte subsets uit in plaats van "alles"
- Live uitvoeringen gebruiken reeds geëxporteerde API-sleutels en voorbereide authenticatieprofielen.
- Standaard isoleren live uitvoeringen nog steeds `HOME` en kopiëren ze configuratie- en authenticatiemateriaal naar een tijdelijke testhome, zodat unitfixtures je echte `~/.openclaw` niet kunnen wijzigen.
- Stel `OPENCLAW_LIVE_USE_REAL_HOME=1` alleen in wanneer live tests bewust je echte thuismap moeten gebruiken.
- `pnpm test:live` gebruikt standaard een stillere modus: de voortgangsuitvoer van `[live] ...` blijft behouden en Gateway-opstartlogboeken/Bonjour-berichten worden gedempt. Stel `OPENCLAW_LIVE_TEST_QUIET=0` in als je de volledige opstartlogboeken weer wilt zien.
- Rotatie van API-sleutels (providerspecifiek): stel `*_API_KEYS` in met een door komma's/puntkomma's gescheiden indeling of `*_API_KEY_1`, `*_API_KEY_2` (bijvoorbeeld `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) of een override per live uitvoering via `OPENCLAW_LIVE_*_KEY`; tests proberen het opnieuw bij antwoorden vanwege frequentielimieten.
- Voortgangs-/heartbeatuitvoer:
  - Live suites schrijven voortgangsregels naar stderr, zodat langdurige provideraanroepen zichtbaar actief blijven, zelfs wanneer de consolevastlegging van Vitest stil is.
  - `test/vitest/vitest.live.config.ts` schakelt de consoleonderschepping van Vitest uit, zodat voortgangsregels van providers/de Gateway tijdens live uitvoeringen onmiddellijk worden gestreamd.
  - Stem heartbeats voor directe modellen af met `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Stem heartbeats voor de Gateway/probes af met `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Welke suite moet ik uitvoeren?

Gebruik deze beslissingstabel:

- Logica/tests bewerken: voer `pnpm test` uit (en `pnpm test:coverage` als je veel hebt gewijzigd)
- Gateway-netwerken / WS-protocol / koppeling aanpassen: voeg `pnpm test:e2e` toe
- Problemen met "mijn bot is offline" / providerspecifieke fouten / het aanroepen van tools opsporen: voer een beperkte `pnpm test:live` uit

## Live tests (met netwerkverkeer)

Voor de matrix met live modellen, smokes voor CLI-backends, ACP-smokes, de Codex-app-server-
harness en alle live tests voor mediaproviders (Deepgram, BytePlus, ComfyUI,
afbeeldingen, muziek, video, mediaharness) — plus de verwerking van inloggegevens voor live uitvoeringen

- zie [Live suites testen](/nl/help/testing-live). Zie voor de specifieke checklist voor updates en
  pluginvalidatie
  [Updates en plugins testen](/nl/help/testing-updates-plugins).

## Docker-runners (optionele controles voor "werkt in Linux")

Deze Docker-runners zijn verdeeld in twee categorieën:

- Runners voor live modellen: `test:docker:live-models` en `test:docker:live-gateway` voeren binnen de Docker-image van de repository alleen hun bijbehorende live bestand met profielsleutels uit (`src/agents/models.profiles.live.test.ts` en `src/gateway/gateway-models.profiles.live.test.ts`) en koppelen daarbij je lokale configuratiemap, werkruimte en optionele profielomgevingsbestand. De bijbehorende lokale toegangspunten zijn `test:live:models-profiles` en `test:live:gateway-profiles`.
- Docker-live-runners behouden waar nodig hun eigen praktische limieten:
  `test:docker:live-models` gebruikt standaard de zorgvuldig geselecteerde, ondersteunde set met een hoog signaalgehalte, en
  `test:docker:live-gateway` gebruikt standaard `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` en
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Stel `OPENCLAW_LIVE_MAX_MODELS`
  of de Gateway-omgevingsvariabelen in wanneer je expliciet een kleinere limiet of grotere scan wilt.
- `test:docker:all` bouwt de live Docker-image eenmaal via `test:docker:live-build`, verpakt OpenClaw eenmaal als een npm-tarball via `scripts/package-openclaw-for-docker.mjs` en bouwt/hergebruikt vervolgens twee `scripts/e2e/Dockerfile`-images. De kale image is alleen de Node/Git-runner voor installatie-, update- en plugin-afhankelijkheidslanes; die lanes koppelen de vooraf gebouwde tarball. De functionele image installeert dezelfde tarball in `/app` voor functionaliteitslanes van de gebouwde app. Definities van Docker-lanes staan in `scripts/lib/docker-e2e-scenarios.mjs`; de plannerlogica staat in `scripts/lib/docker-e2e-plan.mjs`; `scripts/test-docker-all.mjs` voert het geselecteerde plan uit. Het aggregaat gebruikt een gewogen lokale planner: `OPENCLAW_DOCKER_ALL_PARALLELISM` bepaalt het aantal processlots, terwijl resourcelimieten voorkomen dat zware live-, npm-installatie- en multiservicelanes allemaal tegelijk starten. Als één lane zwaarder is dan de actieve limieten, kan de planner deze toch starten wanneer de pool leeg is en laat deze vervolgens alleen draaien totdat er weer capaciteit beschikbaar is. De standaardwaarden zijn 10 slots, `OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`, `OPENCLAW_DOCKER_ALL_NPM_LIMIT=5` en `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7`; pas `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` of `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT` (en andere `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT`-overrides) alleen aan wanneer de Docker-host meer vrije capaciteit heeft. De runner voert standaard een Docker-preflight uit, verwijdert verouderde OpenClaw E2E-containers, toont elke 30 seconden de status, slaat timinggegevens van geslaagde lanes op in `.artifacts/docker-tests/lane-timings.json` en gebruikt die timinggegevens om bij latere uitvoeringen langere lanes eerst te starten. Gebruik `OPENCLAW_DOCKER_ALL_DRY_RUN=1` om het gewogen lanemanifest weer te geven zonder Docker te bouwen of uit te voeren, of `node scripts/test-docker-all.mjs --plan-json` om het CI-plan voor geselecteerde lanes, pakket-/imagevereisten en inloggegevens weer te geven.
- `Package Acceptance` is de GitHub-eigen pakketpoort voor "werkt deze installeerbare tarball als product?" Deze bepaalt één kandidaatpakket uit `source=npm`, `source=ref`, `source=url`, `source=trusted-url` of `source=artifact`, uploadt het als `package-under-test` en voert vervolgens de herbruikbare Docker E2E-lanes uit met precies die tarball, in plaats van de geselecteerde ref opnieuw te verpakken. Profielen zijn gerangschikt op breedte: `smoke`, `package`, `product` en `full` (plus `custom` voor een expliciete lanelijst). Zie [Updates en plugins testen](/nl/help/testing-updates-plugins) voor het pakket-/update-/plugincontract, de survivormatrix voor gepubliceerde upgrades, releasestandaarden en fouttriage.
- Bouw- en releasecontroles voeren `scripts/check-cli-bootstrap-imports.mjs` uit na tsdown. De controle doorloopt de statisch gebouwde graaf vanaf `dist/entry.js` en `dist/cli/run-main.js` en mislukt als die bootstrapgraaf vóór opdrachtverwerking statisch een extern pakket importeert (Commander, prompt-UI, undici, logboekregistratie en vergelijkbare zware opstartafhankelijkheden tellen allemaal mee); daarnaast beperkt deze het gebundelde Gateway-uitvoeringssegment tot 70 KB en weigert deze statische imports vanuit dat segment van bekende koude Gateway-paden (`control-ui-assets`, `diagnostic-stability-bundle`, `onboard-helpers`, `process-respawn`, `restart-sentinel`, `server-close`, `server-reload-handlers`). `scripts/release-check.ts` voert afzonderlijk smoketests uit op de verpakte CLI met `--help`, `onboard --help`, `doctor --help`, `status --json --timeout 1`, `config schema` en `models list --provider openai`.
- Verouderde compatibiliteit van Package Acceptance is beperkt tot `2026.4.25` (`2026.4.25-beta.*` inbegrepen). Tot en met die grens tolereert de harness alleen hiaten in metadata van geleverde pakketten: ontbrekende privé-QA-inventarisvermeldingen, ontbrekende `gateway install --wrapper`, ontbrekende patchbestanden in de van de tarball afgeleide gitfixture, ontbrekende persistente `update.channel`, verouderde locaties van plugininstallatierecords, ontbrekende persistentie van marketplace-installatierecords en migratie van configuratiemetadata tijdens `plugins update`. Voor pakketten na `2026.4.25` zijn die paden strikte fouten.
- Containersmoke-runners: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:npm-onboard-channel-agent`, `test:docker:release-user-journey`, `test:docker:release-typed-onboarding`, `test:docker:release-media-memory`, `test:docker:release-upgrade-user-journey`, `test:docker:release-plugin-marketplace`, `test:docker:skill-install`, `test:docker:update-channel-switch`, `test:docker:upgrade-survivor`, `test:docker:published-upgrade-survivor`, `test:docker:session-runtime-context`, `test:docker:agents-delete-shared-workspace`, `test:docker:gateway-network`, `test:docker:browser-cdp-snapshot`, `test:docker:mcp-channels`, `test:docker:agent-bundle-mcp-tools`, `test:docker:cron-mcp-cleanup`, `test:docker:plugins`, `test:docker:plugin-update`, `test:docker:plugin-lifecycle-matrix` en `test:docker:config-reload` starten een of meer echte containers op en verifiëren integratiepaden op hoger niveau.
- Docker/Bash E2E-lanes die de verpakte OpenClaw-tarball via `scripts/lib/openclaw-e2e-instance.sh` installeren, beperken `npm install` tot `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT` (standaard `600s`; stel `0` in om de wrapper voor foutopsporing uit te schakelen).

De Docker-runners voor live modellen koppelen bovendien alleen de benodigde CLI-authenticatiehomes
(of alle ondersteunde homes wanneer de uitvoering niet is beperkt) en kopiëren deze vervolgens vóór de uitvoering naar de
containerhome, zodat OAuth van externe CLI's tokens kan vernieuwen
zonder het authenticatiearchief op de host te wijzigen:

- Directe modellen: `pnpm test:docker:live-models` (script: `scripts/test-live-models-docker.sh`)
- ACP-koppelingssmoke: `pnpm test:docker:live-acp-bind` (script: `scripts/test-live-acp-bind-docker.sh`; omvat standaard Claude, Codex en Gemini, met strikte dekking voor Droid/OpenCode via `pnpm test:docker:live-acp-bind:droid` en `pnpm test:docker:live-acp-bind:opencode`)
- Smoke voor CLI-backend: `pnpm test:docker:live-cli-backend` (script: `scripts/test-live-cli-backend-docker.sh`)
- Smoke voor Codex-app-serverharness: `pnpm test:docker:live-codex-harness` (script: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + ontwikkelagent: `pnpm test:docker:live-gateway` (script: `scripts/test-live-gateway-models-docker.sh`)
- Waarneembaarheidssmokes: `pnpm qa:otel:smoke`, `pnpm qa:prometheus:smoke` en `pnpm qa:observability:smoke` zijn privé-QA-lanes voor broncodecheckouts. Ze maken bewust geen deel uit van Docker-releaselanes voor pakketten, omdat de npm-tarball QA Lab weglaat.
- Open WebUI-live-smoke: `pnpm test:docker:openwebui` (script: `scripts/e2e/openwebui-docker.sh`)
- Onboardingwizard (TTY, volledige basisstructuur): `pnpm test:docker:onboard` (script: `scripts/e2e/onboard-docker.sh`)
- Npm-tarballsmoke voor onboarding/kanaal/agent: `pnpm test:docker:npm-onboard-channel-agent` installeert de verpakte OpenClaw-tarball globaal in Docker, configureert standaard OpenAI via onboarding met een omgevingsverwijzing plus Telegram, voert doctor uit en voert één gesimuleerde OpenAI-agentbeurt uit. Hergebruik een vooraf gebouwde tarball met `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz`, sla het opnieuw bouwen op de host over met `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0` of wissel van kanaal met `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` of `OPENCLAW_NPM_ONBOARD_CHANNEL=slack`.

- Smoke-test van de releasegebruikersreis: `pnpm test:docker:release-user-journey` installeert de verpakte OpenClaw-tarball globaal in een schone Docker-home, voert onboarding uit, configureert een gesimuleerde OpenAI-provider, voert een agentbeurt uit, installeert/verwijdert externe plugins, configureert ClickClack met een lokale fixture, verifieert uitgaande/inkomende berichten, herstart Gateway en voert doctor uit.
- Smoke-test van getypte release-onboarding: `pnpm test:docker:release-typed-onboarding` installeert de verpakte tarball, stuurt `openclaw onboard` aan via een echte TTY, configureert OpenAI als een env-ref-provider, verifieert dat de onbewerkte sleutel niet wordt opgeslagen en voert een gesimuleerde agentbeurt uit.
- Smoke-test van media/geheugen voor de release: `pnpm test:docker:release-media-memory` installeert de verpakte tarball en verifieert beeldbegrip op basis van een PNG-bijlage, uitvoer van OpenAI-compatibele beeldgeneratie, herinnering via geheugenzoekopdrachten en het behoud van herinneringen na een herstart van Gateway.
- Smoke-test van de gebruikersreis bij een release-upgrade: `pnpm test:docker:release-upgrade-user-journey` installeert standaard de nieuwste gepubliceerde basisversie die ouder is dan de kandidaat-tarball, configureert provider-/plugin-/ClickClack-status op het gepubliceerde pakket, voert een upgrade uit naar de kandidaat-tarball en doorloopt vervolgens opnieuw de kernreis voor agent/plugin/kanaal. Als er geen oudere gepubliceerde basisversie bestaat, wordt de kandidaatversie opnieuw gebruikt. Overschrijf de basisversie met `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>`.
- Smoke-test van de pluginmarktplaats voor de release: `pnpm test:docker:release-plugin-marketplace` installeert vanuit een lokale fixturemarktplaats, werkt de geïnstalleerde plugin bij, verwijdert deze en verifieert dat de plugin-CLI verdwijnt en de installatiemetagegevens zijn opgeschoond.
- Smoke-test voor installatie van Skills: `pnpm test:docker:skill-install` installeert de verpakte OpenClaw-tarball globaal in Docker, schakelt installaties van geüploade archieven uit in de configuratie, haalt via zoeken de slug van de huidige live ClawHub-skill op, installeert deze met `openclaw skills install` en verifieert de geïnstalleerde skill plus de oorsprongs-/vergrendelingsmetagegevens van `.clawhub`.
- Smoke-test voor wisselen van updatekanaal: `pnpm test:docker:update-channel-switch` installeert de verpakte OpenClaw-tarball globaal in Docker, schakelt over van pakket `stable` naar git `dev`, verifieert het opgeslagen kanaal en de werking van de plugin na de update, schakelt vervolgens terug naar pakket `stable` en controleert de updatestatus.
- Smoke-test voor overleving na upgrade: `pnpm test:docker:upgrade-survivor` installeert de verpakte OpenClaw-tarball over een vervuilde fixture van een oude gebruiker met agents, kanaalconfiguratie, plugintoelatingslijsten, verouderde status van pluginafhankelijkheden en bestaande werkruimte-/sessiebestanden. De test voert een pakketupdate plus niet-interactieve doctor uit zonder live provider- of kanaalsleutels, start vervolgens een Gateway op loopback en controleert het behoud van configuratie/status plus de budgetten voor opstarten/status.
- Smoke-test voor overleving na gepubliceerde upgrade: `pnpm test:docker:published-upgrade-survivor` installeert standaard `openclaw@latest`, vult realistische bestanden van een bestaande gebruiker in, configureert die basisversie met een ingebouwd opdrachtrecept, valideert de resulterende configuratie, werkt die gepubliceerde installatie bij naar de kandidaat-tarball, voert niet-interactieve doctor uit, schrijft `.artifacts/upgrade-survivor/summary.json`, start vervolgens een Gateway op loopback en controleert geconfigureerde intenties, statusbehoud, opstarten, `/healthz`, `/readyz` en RPC-statusbudgetten. Overschrijf één basisversie met `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`, laat de geaggregeerde planner exacte lokale basisversies uitbreiden met `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS`, zoals `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15`, en breid probleemvormige fixtures uit met `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS`, zoals `reported-issues`; de verzameling gemelde problemen bevat `configured-plugin-installs` voor automatisch herstel van de installatie van externe OpenClaw-plugins. Package Acceptance stelt deze beschikbaar als `published_upgrade_survivor_baseline`, `published_upgrade_survivor_baselines` en `published_upgrade_survivor_scenarios`, herleidt metabasisversietokens zoals `last-stable-4` of `all-since-2026.4.23`, en Full Release Validation breidt de pakketpoort voor release-soak uit naar `last-stable-4 2026.4.23 2026.5.2 2026.4.15` plus `reported-issues`.
- Smoke-test voor sessieruntimecontext: `pnpm test:docker:session-runtime-context` verifieert het opslaan van transcripties van verborgen runtimecontext plus doctor-herstel van getroffen gedupliceerde vertakkingen voor het herschrijven van prompts.
- Smoke-test voor globale Bun-installatie: `bash scripts/e2e/bun-global-install-smoke.sh` verpakt de huidige bronstructuur, installeert deze met `bun install -g` in een geïsoleerde home en verifieert dat `openclaw infer image providers --json` gebundelde beeldproviders retourneert in plaats van vast te lopen. Gebruik een vooraf gebouwde tarball opnieuw met `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz`, sla de hostbuild over met `OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` of kopieer `dist/` uit een gebouwd Docker-image met `OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local`.
- Docker-smoke-test voor het installatieprogramma: `bash scripts/test-install-sh-docker.sh` deelt één npm-cache tussen de root-, update- en direct-npm-containers. De updatesmoke-test gebruikt standaard npm `latest` als stabiele basisversie vóór de upgrade naar de kandidaat-tarball. Overschrijf dit lokaal met `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22` of op GitHub met de invoer `update_baseline_version` van de Install Smoke-workflow. Controles van het installatieprogramma zonder root behouden een geïsoleerde npm-cache, zodat cachevermeldingen die eigendom zijn van root het installatgedrag op gebruikersniveau niet maskeren. Stel `OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache` in om de root-/update-/direct-npm-cache opnieuw te gebruiken bij lokale herhalingen.
- Install Smoke-CI slaat de dubbele globale direct-npm-update over met `OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1`; voer het script lokaal zonder die omgevingsvariabele uit wanneer dekking van directe `npm install -g` nodig is.
- CLI-smoke-test voor agents die een gedeelde werkruimte verwijderen: `pnpm test:docker:agents-delete-shared-workspace` (script: `scripts/e2e/agents-delete-shared-workspace-docker.sh`) bouwt standaard het image uit het Dockerfile in de hoofdmap, vult twee agents met één werkruimte in een geïsoleerde container-home, voert `agents delete --json` uit en verifieert geldige JSON plus het gedrag voor het behouden van de werkruimte. Gebruik het install-smoke-image opnieuw met `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1`.
- Gateway-netwerken en hostlevenscyclus: `pnpm test:docker:gateway-network` (script: `scripts/e2e/gateway-network-docker.sh`) behoudt de LAN-WebSocket-smoke-test voor authenticatie/gezondheid met twee containers en gebruikt vervolgens Admin HTTP op loopback om prepare-afscherming, toegang met behouden besturing, herstel na hervatting en een voorbereide stop/start binnen dezelfde container aan te tonen. De herstartcontrole moet zijn voltooid voordat de oorspronkelijke lease verloopt, verifieert dat de onderbrekingsstatus proceslokaal is terwijl opgeslagen Gateway-configuratie en containeridentiteit behouden blijven, en produceert machineleesbare JSON met fasetijden.
- Smoke-test voor CDP-snapshots van de browser: `pnpm test:docker:browser-cdp-snapshot` (script: `scripts/e2e/browser-cdp-snapshot-docker.sh`) bouwt het E2E-image uit de broncode plus een Chromium-laag, start Chromium met onbewerkte CDP, voert `browser doctor --deep` uit en verifieert dat CDP-rolsnapshots link-URL's, via de cursor tot klikbaar gepromoveerde elementen, iframe-verwijzingen en framemetagegevens omvatten.
- Regressie voor minimale redenering bij OpenAI Responses-web_search: `pnpm test:docker:openai-web-search-minimal` (script: `scripts/e2e/openai-web-search-minimal-docker.sh`) voert een gesimuleerde OpenAI-server uit via Gateway, verifieert dat `web_search` `reasoning.effort` verhoogt van `minimal` naar `low`, forceert vervolgens de afwijzing door het providerschema en controleert of de onbewerkte details in de Gateway-logboeken verschijnen.
- MCP-kanaalbrug (vooraf gevulde Gateway + stdio-brug + smoke-test voor onbewerkte Claude-notificatieframes): `pnpm test:docker:mcp-channels` (script: `scripts/e2e/mcp-channels-docker.sh`)
- MCP-tools van de OpenClaw-bundel (echte stdio-MCP-server + smoke-test voor toestaan/weigeren in een ingebed OpenClaw-profiel): `pnpm test:docker:agent-bundle-mcp-tools` (script: `scripts/e2e/agent-bundle-mcp-tools-docker.sh`)
- MCP-opschoning voor Cron/subagent (echte Gateway + opruimen van stdio-MCP-kindproces na geïsoleerde cron- en eenmalige subagentruns): `pnpm test:docker:cron-mcp-cleanup` (script: `scripts/e2e/cron-mcp-cleanup-docker.sh`)
- Plugins (smoke-test voor installatie/update via lokaal pad, `file:`, npm-register met gehoste afhankelijkheden, onjuiste npm-pakketmetagegevens, bewegende git-verwijzingen, uitgebreide ClawHub-testset, marktplaatsupdates en inschakelen/inspecteren van de Claude-bundel): `pnpm test:docker:plugins` (script: `scripts/e2e/plugins-docker.sh`)
  Stel `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` in om het ClawHub-blok over te slaan, of overschrijf het standaardpaar van uitgebreid pakket/runtime met `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` en `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID`. Zonder `OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL` gebruikt de test een hermetische lokale ClawHub-fixtureserver.
- Smoke-test voor ongewijzigde pluginupdate: `pnpm test:docker:plugin-update` (script: `scripts/e2e/plugin-update-unchanged-docker.sh`)
- Smoke-test voor de matrix van de pluginlevenscyclus: `pnpm test:docker:plugin-lifecycle-matrix` installeert de verpakte OpenClaw-tarball in een kale container, installeert een npm-plugin, schakelt deze in/uit, voert via een lokaal npm-register upgrades en downgrades uit, verwijdert de geïnstalleerde code en verifieert vervolgens dat verwijdering nog steeds verouderde status opruimt, terwijl RSS-/CPU-metrieken voor elke levenscyclusfase worden geregistreerd.
- Smoke-test voor metagegevens bij herladen van configuratie: `pnpm test:docker:config-reload` (script: `scripts/e2e/config-reload-source-docker.sh`)
- Plugins: `pnpm test:docker:plugins` omvat smoke-tests voor installatie/update via lokaal pad, `file:`, npm-register met gehoste afhankelijkheden, bewegende git-verwijzingen, ClawHub-fixtures, marktplaatsupdates en inschakelen/inspecteren van de Claude-bundel. `pnpm test:docker:plugin-update` omvat gedrag bij ongewijzigde updates voor geïnstalleerde plugins. `pnpm test:docker:plugin-lifecycle-matrix` omvat installatie, inschakelen, uitschakelen, upgraden, downgraden en verwijderen bij ontbrekende code van een npm-plugin, met bijgehouden resources.

Om het gedeelde functionele image vooraf te bouwen en handmatig opnieuw te gebruiken:

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

Suitespecifieke image-overschrijvingen zoals `OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` hebben nog steeds voorrang wanneer ze zijn ingesteld. Wanneer `OPENCLAW_SKIP_DOCKER_BUILD=1` naar een extern gedeeld image verwijst, halen de scripts dit op als het nog niet lokaal aanwezig is. De QR- en Docker-tests voor het installatieprogramma behouden hun eigen Dockerfiles, omdat ze pakket-/installatiegedrag valideren in plaats van de gedeelde runtime van de gebouwde app.

De Docker-runners voor livemodellen koppelen de huidige checkout ook alleen-lezen aan
en plaatsen deze in een tijdelijke werkmap binnen de container. Hierdoor blijft het
runtime-image klein, terwijl Vitest nog steeds wordt uitgevoerd op jouw exacte lokale
broncode/configuratie. De voorbereidingsstap slaat grote uitsluitend lokale caches en app-builduitvoer
over, zoals `.pnpm-store`, `.worktrees`, `__openclaw_vitest__` en
applokale `.build`- of Gradle-uitvoermappen, zodat Docker-liveruns niet
minutenlang machinespecifieke artefacten kopiëren. Ze stellen ook
`OPENCLAW_SKIP_CHANNELS=1` in, zodat live Gateway-probes geen echte
Telegram-/Discord-/enzovoort-kanaalworkers in de container starten.
`test:docker:live-models` voert nog steeds `pnpm test:live` uit, dus geef ook
`OPENCLAW_LIVE_GATEWAY_*` door wanneer je de live Gateway-dekking in die Docker-lane
moet beperken of uitsluiten.

`test:docker:openwebui` is een compatibiliteitssmoke-test op hoger niveau: deze start een
OpenClaw Gateway-container met de OpenAI-compatibele HTTP-eindpunten ingeschakeld,
start een vastgezette Open WebUI-container die met die Gateway is verbonden, meldt zich aan via
Open WebUI, verifieert dat `/api/models` `openclaw/default` beschikbaar stelt en verzendt vervolgens een
echt chatverzoek via de `/api/chat/completions`-proxy van Open WebUI. Stel
`OPENWEBUI_SMOKE_MODE=models` in voor CI-controles van het releasepad die moeten stoppen
na aanmelding bij Open WebUI en modeldetectie, zonder te wachten op de voltooiing door een livemodel.
De eerste run kan merkbaar langzamer zijn, omdat Docker mogelijk het Open WebUI-image moet
ophalen en Open WebUI mogelijk zijn eigen
cold-startconfiguratie moet voltooien. Deze lane verwacht een bruikbare sleutel voor een livemodel, aangeleverd via
de procesomgeving, voorbereide authenticatieprofielen of een expliciete
`OPENCLAW_PROFILE_FILE`. Geslaagde runs drukken een kleine JSON-payload af, zoals
`{ "ok": true, "model": "openclaw/default", ... }`.

`test:docker:mcp-channels` is opzettelijk deterministisch en vereist geen
echt Telegram-, Discord- of iMessage-account. De test start een vooraf gevulde Gateway-container,
start een tweede container die `openclaw mcp serve` voortbrengt en
verifieert vervolgens de detectie van gerouteerde gesprekken, het lezen van transcripties, metagegevens van
bijlagen, gedrag van de livegebeurteniswachtrij, routering van uitgaande verzending en Claude-achtige
kanaal- en toestemmingsmeldingen via de echte stdio-MCP-brug. De
meldingscontrole inspecteert de onbewerkte stdio-MCP-frames rechtstreeks, zodat de smoke-test
valideert wat de brug daadwerkelijk produceert, en niet alleen wat een specifieke client-SDK
toevallig beschikbaar stelt.

`test:docker:agent-bundle-mcp-tools` is deterministisch en heeft geen actieve
modelsleutel nodig. Het bouwt de Docker-image van de repo, start een echte stdio-MCP-
testserver in de container, maakt die server beschikbaar via de
ingebouwde MCP-runtime van de OpenClaw-bundel, voert de tool uit en verifieert vervolgens
dat `coding` en `messaging` de `bundle-mcp`-tools behouden, terwijl `minimal` en
`tools.deny: ["bundle-mcp"]` ze filteren.

`test:docker:cron-mcp-cleanup` is deterministisch en heeft geen actieve
modelsleutel nodig. Het start een vooraf gevulde Gateway met een echte stdio-MCP-testserver,
voert een geïsoleerde cron-beurt en een eenmalige onderliggende `sessions_spawn`-beurt uit en
verifieert vervolgens dat het onderliggende MCP-proces na elke uitvoering wordt afgesloten.

Handmatige ACP-rooktest voor threads in gewone taal (niet in CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Bewaar dit script voor regressie- en foutopsporingsworkflows. Het kan opnieuw nodig zijn voor de validatie van ACP-threadroutering, dus verwijder het niet.

Nuttige omgevingsvariabelen:

- `OPENCLAW_CONFIG_DIR=...` (standaard: `~/.openclaw`) gekoppeld aan `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (standaard: `~/.openclaw/workspace`) gekoppeld aan `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` gekoppeld en ingelezen voordat tests worden uitgevoerd
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` om alleen omgevingsvariabelen te verifiëren die uit `OPENCLAW_PROFILE_FILE` zijn ingelezen, met tijdelijke configuratie-/werkruimtemappen en zonder externe CLI-authenticatiekoppelingen
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (standaard: `~/.cache/openclaw/docker-cli-tools`, tenzij de uitvoering al een CI-/beheerde koppelingsmap gebruikt) gekoppeld aan `/home/node/.npm-global` voor gecachte CLI-installaties binnen Docker
- Externe CLI-authenticatiemappen/-bestanden onder `$HOME` worden alleen-lezen gekoppeld onder `/host-auth...` en vervolgens naar `/home/node/...` gekopieerd voordat de tests beginnen
  - Standaardmappen (gebruikt wanneer de uitvoering niet tot specifieke providers is beperkt): `.factory`, `.gemini`, `.minimax`
  - Standaardbestanden: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Beperkte provideruitvoeringen koppelen alleen de benodigde mappen/bestanden die uit `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` zijn afgeleid
  - Overschrijf dit handmatig met `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` of een door komma's gescheiden lijst zoals `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` om de uitvoering te beperken
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` om providers in de container te filteren
- `OPENCLAW_SKIP_DOCKER_BUILD=1` om een bestaande `openclaw:local-live`-image opnieuw te gebruiken voor herhaalde uitvoeringen waarvoor geen nieuwe build nodig is
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` om te verzekeren dat referenties uit de profielopslag komen (niet uit de omgeving)
- `OPENCLAW_OPENWEBUI_MODEL=...` om het model te kiezen dat door de Gateway beschikbaar wordt gesteld voor de Open WebUI-rooktest
- `OPENCLAW_OPENWEBUI_PROMPT=...` om de prompt voor de nonce-controle van de Open WebUI-rooktest te overschrijven
- `OPENWEBUI_IMAGE=...` om de vastgezette Open WebUI-imagetag te overschrijven

## Documentatiecontrole

Voer documentatiecontroles uit na wijzigingen aan de documentatie: `pnpm check:docs`.
Voer de volledige Mintlify-ankervalidatie uit wanneer je ook controles van paginakoppen nodig hebt: `pnpm docs:check-links:anchors`.

## Offlineregressie (CI-veilig)

Dit zijn regressies van de "echte pijplijn" zonder echte providers:

- Toolaanroepen via de Gateway (nagebootste OpenAI, echte Gateway + agentlus): `src/gateway/gateway.test.ts` (geval: "voert een nagebootste OpenAI-toolaanroep end-to-end uit via de agentlus van de Gateway")
- Gateway-wizard (WS `wizard.start`/`wizard.next`, schrijft configuratie + dwingt authenticatie af): `src/gateway/gateway.test.ts` (geval: "voert de wizard via ws uit en schrijft de configuratie van het authenticatietoken")

## Betrouwbaarheidsevaluaties van agents (Skills)

We hebben al enkele CI-veilige tests die functioneren als "betrouwbaarheidsevaluaties van agents":

- Nagebootste toolaanroepen via de echte Gateway + agentlus (`src/gateway/gateway.test.ts`).
- End-to-end-wizardflows die de sessiekoppeling en configuratie-effecten valideren (`src/gateway/gateway.test.ts`).

Wat nog ontbreekt voor Skills (zie [Skills](/nl/tools/skills)):

- **Besluitvorming:** kiest de agent de juiste Skill wanneer Skills in de prompt worden vermeld (of vermijdt de agent irrelevante Skills)?
- **Naleving:** leest de agent vóór gebruik `SKILL.md` en volgt deze de vereiste stappen/argumenten?
- **Workflowcontracten:** scenario's met meerdere beurten die de toolvolgorde, het meenemen van sessiegeschiedenis en sandboxgrenzen controleren.

Toekomstige evaluaties moeten eerst deterministisch blijven:

- Een scenariorunner die nagebootste providers gebruikt om toolaanroepen + volgorde, het lezen van Skill-bestanden en sessiekoppeling te controleren.
- Een kleine suite met op Skills gerichte scenario's (gebruiken versus vermijden, toegangscontrole, promptinjectie).
- Optionele live-evaluaties (opt-in, afgeschermd via omgevingsvariabelen), maar pas nadat de CI-veilige suite beschikbaar is.

## Contracttests (vorm van plugins en kanalen)

Contracttests verifiëren dat elke geregistreerde Plugin en elk kanaal voldoet aan
het bijbehorende interfacecontract. Ze doorlopen alle gevonden plugins en voeren een
suite met controles op vorm en gedrag uit. De standaard `pnpm test`-unitlane
slaat deze gedeelde koppelvlak- en rooktestbestanden bewust over; voer de contract-
opdrachten expliciet uit wanneer je gedeelde kanaal- of provideroppervlakken aanraakt.

### Opdrachten

- Alle contracten: `pnpm test:contracts`
- Alleen kanaalcontracten: `pnpm test:contracts:channels`
- Alleen providercontracten: `pnpm test:contracts:plugins`

### Kanaalcontracten

Te vinden in `src/channels/plugins/contracts/*.contract.test.ts`. Huidige
hoofdcategorieën:

- **channel-catalog** - metadata van kanaalcatalogusvermeldingen voor gebundelde kanalen/registerkanalen
- **plugin** (registerondersteund, geshard) - basisvorm van Plugin-registratie
- **surfaces-only** (registerondersteund, geshard) - vormcontroles per oppervlak voor `actions`, `setup`, `status`, `outbound`, `messaging`, `threading`, `directory` en `gateway`
- **session-binding** (registerondersteund) - gedrag van sessiekoppelingen
- **outbound-payload** - structuur en normalisatie van berichtpayloads
- **group-policy** (terugval) - afdwingen van het standaardgroepsbeleid per kanaal
- **threading** (registerondersteund, geshard) - verwerking van thread-id's
- **directory** (registerondersteund, geshard) - directory-/roster-API
- **registry** en **plugins-core.\*** - internals voor het register van kanaalplugins, de loader en autorisatie voor het schrijven van configuratie

Helpers van het harnas voor het vastleggen van inkomende dispatches en uitgaande payloads die door deze
suites worden gebruikt, worden intern beschikbaar gesteld via `src/plugin-sdk/channel-contract-testing.ts`
(uitgesloten van npm, geen openbaar SDK-subpad); er is geen zelfstandig
`inbound.contract.test.ts`-bestand in deze directory.

### Providercontracten

Te vinden in `src/plugins/contracts/*.contract.test.ts`. Huidige categorieën
omvatten:

- **shape** - vorm van het Plugin-manifest, de API en runtime-exports
- **plugin-registration** (+ parallel) - gevallen voor manifestregistratie
- **package-manifest** - vereisten voor pakketmanifesten
- **loader** - installatie-/opruimgedrag van de Plugin-loader
- **registry** - inhoud en opzoekgedrag van het Plugin-contractregister
- **providers** - gedeeld providergedrag voor gebundelde providers, plus providers voor zoeken op het web
- **auth-choice** - metadata en installatiegedrag voor authenticatiekeuzes
- **provider-catalog-deprecation** - verouderde metadata van de providercatalogus
- **wizard.choice-resolution**, **wizard.model-picker**, **wizard.setup-options** - contracten van de providerinstallatiewizard
- **embedding-provider**, **memory-embedding-provider**, **web-fetch-provider**, **tts** - functiespecifieke providercontracten
- **session-actions**, **session-attachments**, **session-entry-projection** - door plugins beheerde contracten voor sessiestatus
- **scheduled-turns** - metadata en tijdstempelgrenzen voor geplande Plugin-beurten
- **host-hooks**, **run-context-lifecycle**, **runtime-import-side-effects**, **runtime-seams** - contracten voor de levenscyclus en importgrenzen van de Plugin-host/runtime
- **extension-runtime-dependencies** - plaatsing van runtime-afhankelijkheden voor extensies

### Wanneer uitvoeren

- Na het wijzigen van plugin-sdk-exports of -subpaden
- Na het toevoegen of wijzigen van een kanaal- of providerplugin
- Na het refactoren van Plugin-registratie of -detectie

Contracttests worden uitgevoerd in CI en vereisen geen echte API-sleutels.

## Regressies toevoegen (richtlijnen)

Wanneer je een probleem met een provider/model oplost dat live is ontdekt:

- Voeg indien mogelijk een CI-veilige regressie toe (nagebootste/gesimuleerde provider, of leg de exacte transformatie van de aanvraagvorm vast)
- Als het probleem inherent alleen live optreedt (snelheidslimieten, authenticatiebeleid), houd de live-test dan beperkt en maak deze opt-in via omgevingsvariabelen
- Richt je bij voorkeur op de kleinste laag die de fout detecteert:
  - fout bij conversie/herhaling van providerverzoeken -> directe modeltest
  - fout in de sessie-/geschiedenis-/toolpijplijn van de Gateway -> live-rooktest van de Gateway of CI-veilige nagebootste Gateway-test
- Beveiliging tegen SecretRef-padtraversal:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` leidt uit registermetadata (`listSecretTargetRegistryEntries()`) één steekproefdoel per SecretRef-klasse af en controleert vervolgens dat exec-id's met traversalsegmenten worden geweigerd.
  - Als je in `src/secrets/target-registry-data.ts` een nieuwe `includeInPlan`-SecretRef-doelfamilie toevoegt, werk dan `classifyTargetClass` in die test bij. De test faalt bewust bij niet-geclassificeerde doel-id's, zodat nieuwe klassen niet ongemerkt kunnen worden overgeslagen.

## Gerelateerd

- [Live testen](/nl/help/testing-live)
- [Updates en plugins testen](/nl/help/testing-updates-plugins)
- [CI](/nl/ci)
