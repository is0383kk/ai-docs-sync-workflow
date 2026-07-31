---
doc-schema-version: 1
read_when:
    - Begrijpen hoe de QA-stack samenhangt
    - qa-lab, qa-channel of een transportadapter uitbreiden
    - QA-scenario's met repositoryondersteuning toevoegen
    - QA-automatisering met een hoger realiteitsgehalte rond het Gateway-dashboard bouwen
summary: 'Overzicht van de QA-stack: qa-lab, qa-channel, door de repository ondersteunde scenario''s, live transporttrajecten, transportadapters en rapportage.'
title: QA-overzicht
x-i18n:
    generated_at: "2026-07-27T05:43:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91c34a50e6197195d57228d92b19caff1785ceaa5d82d7c88a1ec0ed76abd635
    source_path: concepts/qa-e2e-automation.md
    workflow: 16
---

De private QA-stack test OpenClaw op een realistische, kanaalgerichte manier die
met een unit-test niet mogelijk is.

Onderdelen:

- `extensions/qa-channel`: synthetisch berichtkanaal met oppervlakken voor DM's, kanalen, threads,
  reacties, bewerkingen en verwijderingen.
- `extensions/qa-lab`: debugger-UI, QA-bus, scenarioprofielen en live
  transportadapters voor het observeren van het transcript, injecteren van inkomende berichten
  en exporteren van een Markdown-rapport.
- `qa/`: door de repo ondersteunde seed-assets voor de starttaak en basis-QA-
  scenario's.
- [Mantis](/nl/concepts/mantis): live verificatie vóór en na voor bugs waarvoor
  echte transporten, browserschermafbeeldingen, VM-status en PR-bewijs nodig zijn.

## Opdrachtoppervlak

Elke QA-flow wordt uitgevoerd onder `pnpm openclaw qa <subcommand>`. Veel hebben `pnpm qa:*`-
scriptaliassen; beide vormen werken.

| Opdracht                                             | Doel                                                                                                                                                                                                                                                             |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | Gebundelde QA-zelfcontrole zonder `--qa-profile`; door taxonomie ondersteunde runner voor volwassenheidsprofielen met `--qa-profile smoke-ci`, `--qa-profile release` of `--qa-profile all`.                                                                                                  |
| `qa suite`                                          | Voer door de repo ondersteunde scenario's uit tegen de QA-Gateway-lane. `--runner multipass` gebruikt een tijdelijke Linux-VM in plaats van de host.                                                                                                                                         |
| `qa coverage`                                       | Druk de YAML-inventaris voor scenariodekking af (`--json` voor machine-uitvoer; `--match <query>` om scenario's voor aangepast gedrag te vinden; `--tools` voor dekking van runtime-toolfixtures).                                                                                  |
| `qa parity-report`                                  | Vergelijk twee `qa-suite-summary.json`-bestanden voor een pariteitsgate op de modelas, of gebruik `--runtime-axis --token-efficiency` om rapporten over runtimepariteit en tokenefficiëntie van Codex versus OpenClaw te schrijven.                                                                          |
| `qa confidence-report`                              | Classificeer QA-bewijsartefacten aan de hand van een manifest in een betrouwbaarheidsrapport zonder onbekende waarden.                                                                                                                                                                               |
| `qa confidence-self-test`                           | Schrijf vooraf ingevulde canaries voor negatieve controles die aantonen dat de betrouwbaarheidsgate afwijkingen detecteert.                                                                                                                                                                                   |
| `qa jsonl-replay`                                   | Speel samengestelde JSONL-transcripten opnieuw af via de replay-harness voor runtimepariteit.                                                                                                                                                                                         |
| `qa character-eval`                                 | Voer het karakter-QA-scenario uit met meerdere live modellen en een beoordeeld rapport. Zie [Rapportage](#reporting).                                                                                                                                                        |
| `qa manual`                                         | Voer een eenmalige prompt uit tegen de geselecteerde provider-/modellane.                                                                                                                                                                                                      |
| `qa ui`                                             | Start de QA-debugger-UI en lokale QA-bus (alias: `pnpm qa:lab:ui`).                                                                                                                                                                                                |
| `qa docker-build-image`                             | Bouw de vooraf samengestelde QA-Docker-image.                                                                                                                                                                                                                                 |
| `qa docker-scaffold`                                | Schrijf een docker-compose-scaffold voor het QA-dashboard en de Gateway-lane.                                                                                                                                                                                                |
| `qa up`                                             | Bouw de QA-site, start de door Docker ondersteunde stack en druk de URL af (alias: `pnpm qa:lab:up`; de `:fast`-variant voegt `--use-prebuilt-image --bind-ui-dist --skip-ui-build` toe).                                                                                              |
| `qa aimock`                                         | Start alleen de AIMock-providerserver.                                                                                                                                                                                                                              |
| `qa mock-openai`                                    | Start alleen de scenariobewuste `mock-openai`-providerserver.                                                                                                                                                                                                        |
| `qa credentials doctor` / `add` / `list` / `remove` | Beheer de gedeelde Convex-pool met inloggegevens.                                                                                                                                                                                                                           |
| `qa discord`                                        | Live transportlane tegen een echt privé-Discord-guildkanaal.                                                                                                                                                                                                   |
| `qa matrix`                                         | QA Lab Matrix-profielen tegen een tijdelijke Tuwunel-homeserver. Zie [Matrix-smokelanes](#matrix-smoke-lanes).                                                                                                                                                      |
| `qa slack`                                          | Live transportlane tegen een echt privé-Slack-kanaal.                                                                                                                                                                                                           |
| `qa telegram`                                       | Live transportlane tegen een echte privé-Telegram-groep.                                                                                                                                                                                                          |
| `qa whatsapp`                                       | Live transportlane tegen echte WhatsApp Web-accounts.                                                                                                                                                                                                             |
| `qa mantis`                                         | Verificatierunner voor vóór en na bij live transportbugs, met bewijs via Discord-statusreacties, Crabbox-desktop-/browsersmoke en Slack-in-VNC-smoke. Zie [Mantis](/nl/concepts/mantis) en [Runbook voor Mantis Slack Desktop](/nl/concepts/mantis-slack-desktop-runbook). |

### Door profielen ondersteunde `qa run`

Door profielen ondersteunde `qa run` leest het lidmaatschap uit `taxonomy.yaml` en stuurt
vervolgens de opgeloste scenario's door via `qa suite`. `--surface` en `--category` filteren
het geselecteerde profiel in plaats van afzonderlijke lanes te definiëren. Het resulterende
`qa-evidence.json` bevat een samenvatting van de profielscorekaart met aantallen voor geselecteerde categorieën
en ontbrekende dekkings-ID's; de afzonderlijke bewijsvermeldingen blijven de
bron van waarheid voor de tests, dekkingsrollen en resultaten. Dekkings-ID's voor taxonomiefuncties
zijn exacte bewijsdoelen, geen aliassen: primaire scenariodekking
voldoet aan overeenkomende ID's, terwijl secundaire dekking adviserend blijft. Elke dekkings-
ID is exact `taxonomy-surface.feature`, met de korte oppervlak-ID uit
`taxonomy.yaml`. Het afzonderlijke veld `surface` van een scenario is een label voor uitvoering/rapportage
(bijvoorbeeld `channel` of `runtime-tool`); het definieert geen taxonomie-
eigenaarschap.

Beknopt bewijs laat `execution` per vermelding weg en stelt `evidenceMode: "slim"` in;
`smoke-ci` gebruikt standaard beknopt bewijs en `--evidence-mode full` herstelt volledige vermeldingen:

```bash
pnpm openclaw qa run \
  --qa-profile smoke-ci \
  --category channels.conversation-routing-and-delivery \
  --provider-mode mock-openai \
  --output-dir .artifacts/qa-e2e/smoke-ci-profile-dispatch
```

Gebruik `smoke-ci` voor deterministisch profielbewijs met mockmodelproviders en
lokale Crabline-providerservers. Gebruik `release` voor Stable/LTS-bewijs tegen
live kanalen. Gebruik `all` alleen voor expliciete bewijsruns voor de volledige taxonomie; dit
selecteert elke actieve volwassenheidscategorie en kan via de `QA
Profile Evidence` GitHub Actions-workflow met `qa_profile=all` worden uitgevoerd. Wanneer een
opdracht ook een OpenClaw-rootprofiel nodig heeft, plaats je het rootprofiel vóór de
QA-opdracht:

```bash
pnpm openclaw --profile work qa run --qa-profile smoke-ci
```

## Operatorflow

De huidige QA-operatorflow is een QA-site met twee vensters:

- Links: Gateway-dashboard (Control UI) met de agent.
- Rechts: QA Lab, met het Slack-achtige transcript en scenarioplan.

Voer deze uit met:

```bash
pnpm qa:lab:up
```

Hiermee wordt de QA-site gebouwd, de door Docker ondersteunde Gateway-lane gestart en
de QA Lab-pagina beschikbaar gemaakt, waar een operator of automatiseringslus de agent een QA-
missie kan geven, echt kanaalgedrag kan observeren en kan vastleggen wat werkte, mislukte of
geblokkeerd bleef.

Voor snellere iteratie van de QA Lab-UI zonder de Docker-image telkens opnieuw te bouwen,
start je de stack met een via bind-mount gekoppelde QA Lab-bundel:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` houdt de Docker-services op een vooraf gebouwde image en
koppelt `extensions/qa-lab/web/dist` via een bind-mount aan de `qa-lab`-container.
`qa:lab:watch` bouwt die bundel opnieuw bij wijzigingen en de browser wordt automatisch opnieuw geladen
wanneer de assethash van QA Lab verandert.

### Observability-smoketests

<Note>
Observability-QA blijft uitsluitend beschikbaar vanuit een source-checkout. Het npm-tarball laat
QA Lab (en `qa-channel`) bewust weg, zodat Docker-releaselanen voor pakketten
geen `qa`-opdrachten uitvoeren. Voer deze uit vanuit een gebouwde source-checkout wanneer
je diagnostische instrumentatie wijzigt.
</Note>

| Alias                                   | Wat het uitvoert                                                                                                                            |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm qa:otel:smoke`                    | Lokale OpenTelemetry-ontvanger plus het scenario `otel-trace-smoke` met `diagnostics-otel` ingeschakeld.                                      |
| `pnpm qa:otel:collector-smoke`          | Dezelfde lane achter een echte OpenTelemetry Collector-Docker-container. Gebruik deze bij wijzigingen aan endpointbedrading of compatibiliteit met de collector/OTLP. |
| `pnpm qa:prometheus:smoke`              | Het scenario `docker-prometheus-smoke` met `diagnostics-prometheus` ingeschakeld.                                                           |
| `pnpm qa:observability:smoke`           | `qa:otel:smoke` gevolgd door `qa:prometheus:smoke`.                                                                                      |
| `pnpm qa:observability:collector-smoke` | `qa:otel:collector-smoke` gevolgd door `qa:prometheus:smoke`.                                                                            |

`qa:otel:smoke` start een lokale OTLP/HTTP-ontvanger, voert een minimale agentbeurt
voor het QA-kanaal uit en controleert vervolgens of traces, metrische gegevens en logs worden geëxporteerd. Het decodeert
de geëxporteerde protobuf-tracespans en controleert de releasekritieke structuur:
`openclaw.run`, `openclaw.harness.run`, een span voor modelaanroepen volgens de nieuwste
semantische GenAI-conventie, `openclaw.context.assembled` en `openclaw.message.delivery`
moeten allemaal aanwezig zijn. De smoke-test dwingt
`OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` af, dus de span voor modelaanroepen
moet de naam `{gen_ai.operation.name} {gen_ai.request.model}` gebruiken; modelaanroepen
mogen bij geslaagde beurten geen `StreamAbandoned` exporteren; onbewerkte diagnostische
ID's en `openclaw.content.*`-attributen mogen niet in de trace terechtkomen. De scenarioprompt
vraagt het model te antwoorden met een vaste markering en een vaste
geheime tekenreeks achter te houden; de onbewerkte OTLP-payloads mogen geen van beide bevatten, noch de
QA-sessiesleutel die van het scenario-ID is afgeleid. Het schrijft `otel-smoke-summary.json`
naast de artefacten van de QA-suite.

`qa:prometheus:smoke` verifieert dat niet-geverifieerde scrapes worden geweigerd en
controleert vervolgens of de geverifieerde scrape releasekritieke metriekfamilies bevat
zonder promptinhoud, antwoordinhoud, onbewerkte diagnostische identificatoren, authenticatie-
tokens of lokale paden.

### Matrix-smoke-lanes

Voer voor een transportechte Matrix-smoke-lane waarvoor geen referenties
van een modelprovider nodig zijn, het releaseprofiel uit met de deterministische nagebootste OpenAI-provider:

```bash
pnpm openclaw qa matrix --provider-mode mock-openai --profile release
```

Geef voor de live-frontier-providerlane expliciet OpenAI-compatibele referenties
op:

```bash
OPENCLAW_LIVE_OPENAI_KEY="${OPENAI_API_KEY}" \
  pnpm openclaw qa matrix --provider-mode live-frontier --profile release
```

Een gewone `pnpm openclaw qa matrix` voert het volledige profiel `all` uit en gaat door na
scenariofouten. Gebruik `--fail-fast` voor een kortere feedbacklus of herhaal
`--scenario <id>` om afzonderlijke scenario's te selecteren; expliciete scenario-ID's hebben
voorrang op `--profile`.

| Profiel      | Scenario's | Doel                                                                                                                                  |
| ------------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `all`        | 93        | Volledige catalogus (standaard).                                                                                                              |
| `release`    | 2         | Releasekritieke kanaalbasislijn en live herladen van de toelatingslijst.                                                                             |
| `fast`       | 12        | Gerichte dekking voor threads, reacties, goedkeuringen, beleid, botblokkering en versleutelde antwoorden.                                               |
| `transport`  | 50        | Threads, routering van privéberichten/ruimten, automatisch deelnemen, goedkeuringen, reacties, herstarts, beleid voor vermeldingen/toelatingslijsten, bewerkingen en volgorde met meerdere actoren.         |
| `media`      | 7         | Dekking voor afbeeldingen, gegenereerde afbeeldingen, spraak, bijlagen, niet-ondersteunde media en versleutelde media.                                              |
| `e2ee-smoke` | 8         | Minimale dekking voor versleutelde antwoorden, threads, bootstrap, herstel, herstart, redactie en fouten.                                       |
| `e2ee-deep`  | 18        | Statusverlies, back-ups, sleutelherstel, apparaathygiëne en SAS-/QR-/privéberichtverificatie.                                                            |
| `e2ee-cli`   | 9         | `openclaw matrix encryption setup`, herstelsleutel-, multi-account-, Gateway-retour- en zelfverificatieopdrachten via het harnas. |

Profiellidmaatschap en kanaalvereisten staan bij de declaratieve Matrix-
scenario's onder `qa/scenarios/channels/`. De uitvoering kiest het kanaalstuurprogramma.
Hun live-implementaties staan onder
`extensions/qa-lab/src/live-transports/matrix/scenarios/`.

De adapter maakt in Docker een tijdelijke Tuwunel-homeserver beschikbaar (standaard-
image `ghcr.io/matrix-construct/tuwunel:v1.5.1`, servernaam `matrix-qa.test`,
poort `28008`), registreert tijdelijke gebruikers voor het stuurprogramma, het te testen systeem en de waarnemer, initialiseert de
vereiste ruimten en legt de geredigeerde grens voor verzoeken en antwoorden vast. Vervolgens
voert deze de echte Matrix-Plugin uit binnen een onderliggende QA-Gateway die tot dat transport
is beperkt (geen `qa-channel`) en ruimt de omgeving op.

Algemene opties:

| Vlag                     | Standaard           | Doel                                                                              |
| ------------------------ | ----------------- | ------------------------------------------------------------------------------------ |
| `--profile <profile>`    | `all`             | Selecteer een van de bovenstaande profielen.                                                    |
| `--scenario <id>`        | -                 | Selecteer één scenario; kan worden herhaald.                                                     |
| `--fail-fast`            | uit               | Stop na de eerste mislukte controle of het eerste mislukte scenario.                                       |
| `--allow-failures`       | uit               | Schrijf artefacten zonder een foutieve afsluitcode te retourneren voor scenariofouten.         |
| `--provider-mode <mode>` | `live-frontier`   | Gebruik `mock-openai` voor deterministische verzending of `live-frontier` voor een live-provider. |
| `--model <ref>`          | providerstandaard  | Stel de primaire `provider/model`-referentie in.                                          |
| `--alt-model <ref>`      | providerstandaard  | Stel het alternatieve model in dat wordt gebruikt door scenario's die van model wisselen.                        |
| `--fast`                 | uit               | Schakel de snelle providermodus in waar deze wordt ondersteund.                                           |
| `--output-dir <path>`    | gegenereerd         | Kies de rapportmap; relatieve paden worden ten opzichte van `--repo-root` bepaald.           |
| `--repo-root <path>`     | huidige map | Voer uit vanuit een neutrale werkmap.                                                |
| `--sut-account <id>`     | `sut`             | Selecteer het Matrix-account-ID in de configuratie van de onderliggende Gateway.                            |

Matrix-QA leaset geen gedeelde Matrix-referenties: de adapter maakt
lokaal tijdelijke gebruikers aan en accepteert daarom geen `--credential-source` of
`--credential-role`. Overschrijf de homeserver-image met
`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE`; stem negatieve controles voor het uitblijven van antwoorden af met
`OPENCLAW_QA_MATRIX_NO_REPLY_WINDOW_MS` (standaard `8000`, begrensd op de time-out van het actieve
scenario). De eenmalige opdracht dwingt normaal gesproken een nette afsluiting af nadat
de artefacten zijn weggeschreven, omdat native handles voor Matrix-cryptografie na het opruimen actief kunnen blijven; stel
`OPENCLAW_QA_MATRIX_DISABLE_FORCE_EXIT=1` alleen in voor een direct testharnas
waarbij de opdracht in plaats daarvan moet terugkeren.

Elke uitvoering schrijft de normale QA Lab-artefacten naar de geselecteerde uitvoer-
map: `qa-suite-report.md`, `qa-suite-summary.json` en
`qa-evidence.json`. Voer de weergegeven
`docker compose ... down --remove-orphans`-herstelopdracht uit als het opruimen mislukt. Vergroot
op langzame runners het venster voor het uitblijven van antwoorden; op snelle CI kan een kleiner venster negatieve
controles verkorten.

De scenario's dekken transportgedrag dat unit-tests niet end-to-end kunnen
bewijzen: blokkering op basis van vermeldingen, beleid voor toegestane bots, toelatingslijsten, antwoorden op hoofdniveau en in
threads, routering van privéberichten, afhandeling van reacties, onderdrukking van inkomende bewerkingen, deduplicatie van opnieuw afgespeelde berichten na
een herstart, herstel na onderbreking van de homeserver, levering van goedkeuringsmetadata,
afhandeling van media en bootstrap-/herstel-/verificatiestromen voor Matrix-E2EE. Het
E2EE-CLI-profiel stuurt ook `openclaw matrix encryption setup` en
verificatieopdrachten via dezelfde tijdelijke homeserver aan voordat het
Gateway-antwoorden controleert.

`matrix-room-block-streaming` en `subagent-thread-spawn` blijven beschikbaar via
expliciete selectie met `--scenario`, maar blijven buiten het standaardprofiel `all`.

CI gebruikt hetzelfde opdrachtoppervlak in
`.github/workflows/qa-live-transports-convex.yml`. Geplande uitvoeringen en release-uitvoeringen
voeren de releasescenario's uit. Handmatige `matrix_profile=all`-dispatches waaieren uit over
de profielen `transport`, `media`, `e2ee-smoke`, `e2ee-deep` en `e2ee-cli`;
gerichte dispatches selecteren `fast`, `release` of `transport` in één taak.

### Discord Mantis-scenario's

Discord heeft ook uitsluitend voor Mantis bedoelde opt-in-scenario's om bugs te reproduceren. Gebruik
`--scenario discord-status-reactions-tool-only` voor de expliciete tijdlijn
van statusreacties of `--scenario discord-thread-reply-filepath-attachment`
om een echte Discord-thread te maken en te verifiëren dat `message.thread-reply`
een `filePath`-bijlage behoudt. Deze scenario's blijven buiten de standaard
live Discord-lane omdat het reproductiecontroles voor vóór/na zijn in plaats van
brede smoke-dekking. De Mantis-workflow voor threadbijlagen kan ook een
getuigenvideo van een ingelogde Discord Web-sessie toevoegen wanneer
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` of
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` in de QA-
omgeving is geconfigureerd. Dat kijkersprofiel is uitsluitend bedoeld voor visuele vastlegging; de beslissing
over slagen of mislukken blijft afkomstig van de Discord REST-orakel.

Voor de andere transportechte smoke-lanes:

```bash
pnpm openclaw qa discord
pnpm openclaw qa slack
pnpm openclaw qa telegram
pnpm openclaw qa whatsapp
```

Ze richten zich op een reeds bestaand echt kanaal met twee bots of accounts (stuurprogramma +
te testen systeem). Vereiste omgevingsvariabelen, scenariolijsten, uitvoerartefacten en de Convex-
pool met referenties voor deze vier transporten worden beschreven in de
[QA-referentie voor Discord, Slack, Telegram en WhatsApp](#discord-slack-telegram-and-whatsapp-qa-reference)
hieronder.

### Mantis-runners voor Slack-desktop- en visuele taken

Voer voor een volledige Slack-desktop-VM-uitvoering met VNC-herstel het volgende uit:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Die opdracht leaset een Crabbox-desktop-/browsermachine, voert de live Slack-
lane uit in de VM, opent Slack Web in de VNC-browser, legt het bureaublad vast
en kopieert `slack-qa/`, `slack-desktop-smoke.png` en
`slack-desktop-smoke.mp4` (wanneer video-opname beschikbaar is) terug naar de
Mantis-artifactmap. Crabbox-desktop-/browserleases leveren de opnamehulpmiddelen
en hulppakketten voor browsers/native builds vooraf, zodat het scenario alleen
fallbacks op oudere leases hoeft te installeren. Mantis rapporteert totale
timings en timings per fase in `mantis-slack-desktop-smoke-report.md`, zodat bij trage uitvoeringen
zichtbaar is of de tijd is besteed aan het opwarmen van de lease, het verkrijgen
van aanmeldgegevens, externe configuratie of het kopiëren van artifacts.
Hergebruik `--lease-id <cbx_...>` nadat je handmatig via VNC bent ingelogd bij Slack
Web; hergebruikte leases houden ook de pnpm-storecache van Crabbox warm. De
standaardwaarde `--hydrate-mode source` verifieert vanuit een broncheckout en voert
installatie/build uit binnen de VM. Gebruik `--hydrate-mode prehydrated` alleen wanneer
de hergebruikte externe werkruimte al `node_modules` en een gebouwde
`dist/` bevat; die modus slaat de kostbare installatie-/buildstap over
en stopt veilig met een fout wanneer de werkruimte niet gereed is. Met
`--gateway-setup` laat Mantis een permanente OpenClaw Slack-gateway actief
binnen de VM op poort `38973`; zonder deze optie voert de opdracht de
normale bot-naar-bot-Slack-QA-lane uit en sluit deze af nadat de artifacts zijn
vastgelegd.

Voer de Mantis-modus voor goedkeuringscontrolepunten uit om de native
Slack-goedkeuringsinterface met desktopbewijs aan te tonen:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer
```

Deze modus sluit `--gateway-setup` wederzijds uit. De modus voert de
Slack-goedkeuringsscenario's uit, wijst scenario-id's af die niet over
goedkeuring gaan, wacht bij elke openstaande en afgehandelde goedkeuringsstatus,
rendert het waargenomen Slack-API-bericht naar `approval-checkpoints/<scenario>-pending.png` en
`approval-checkpoints/<scenario>-resolved.png` en mislukt vervolgens als een controlepunt, berichtbewijs,
bevestiging of gerenderde schermafbeelding ontbreekt of leeg is. Koude
CI-leases kunnen in `slack-desktop-smoke.png` nog steeds de Slack-aanmelding tonen; de
afbeeldingen van de goedkeuringscontrolepunten vormen het visuele bewijs voor
deze lane.

De standaarduitvoering met controlepunten behoudt de twee standaardscenario's
voor Slack-goedkeuring. Als je een van de optionele
Codex-goedkeuringsroutes wilt vastleggen, selecteer je die expliciet met
`--scenario slack-codex-approval-exec-native` of `--scenario slack-codex-approval-plugin-native`; Mantis accepteert beide en genereert
hetzelfde paar schermafbeeldingen voor de openstaande/afgehandelde status. De
runner verlengt voor elke geselecteerde Codex-route de deadlines voor
controlepunten en externe opdrachten, zodat de volledige reeks van goedkeuring,
agentvoltooiing en afgehandelde update kan worden voltooid.

De operatorchecklist, opdracht voor het starten van de GitHub-workflow,
overeenkomst voor bewijscommentaar, beslistabel voor de hydrate-modus,
interpretatie van timings en stappen voor foutafhandeling staan in
[Mantis-runbook voor Slack Desktop](/nl/concepts/mantis-slack-desktop-runbook).

Voer voor een desktoptaak in agent-/CV-stijl het volgende uit:

```bash
pnpm openclaw qa mantis visual-task \
  --browser-url https://example.net \
  --expect-text "Example Domain" \
  --vision-model openai/gpt-5.6-luna
```

`visual-task` leaset of hergebruikt een Crabbox-desktop-/browsermachine,
start `crabbox record --while`, bestuurt de zichtbare browser via een geneste
`visual-driver`, legt `visual-task.png` vast, voert `openclaw infer image
describe` uit
op de schermafbeelding wanneer `--vision-mode image-describe` is geselecteerd en schrijft
`visual-task.mp4`, `mantis-visual-task-summary.json`, `mantis-visual-task-driver-result.json` en
`mantis-visual-task-report.md`. Wanneer `--expect-text` is ingesteld, vraagt de
vision-prompt om een gestructureerd JSON-oordeel (`visible`,
`evidence`, `reason`) en slaagt deze alleen wanneer het model
`visible: true` rapporteert met bewijs dat de verwachte tekst aanhaalt; een
`visible: false`-antwoord dat alleen de doeltekst citeert, voldoet nog steeds
niet aan de assertie. Gebruik `--vision-mode metadata` voor een smoke-test zonder
model die de werking van het bureaublad, de browser, schermafbeeldingen en video
aantoont zonder een provider voor beeldbegrip aan te roepen. Een opname is een
vereist artifact voor `visual-task`; als Crabbox geen niet-lege
`visual-task.mp4` opneemt, mislukt de taak zelfs wanneer het visuele
stuurprogramma is geslaagd. Bij een fout behoudt Mantis de lease voor VNC,
tenzij de taak al was geslaagd en `--keep-lease` niet was ingesteld.

### Gezondheidscontrole van de pool met aanmeldgegevens

Voer voordat je gepoolde live-aanmeldgegevens gebruikt het volgende uit:

```bash
pnpm openclaw qa credentials doctor
```

De doctor controleert de Convex-brokeromgeving (`OPENCLAW_QA_CONVEX_SITE_URL`,
`OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`), valideert de endpointinstellingen, rapporteert voor
`OPENCLAW_QA_CONVEX_SECRET_CI` en `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` alleen de status ingesteld/ontbreekt en
verifieert de bereikbaarheid voor beheer/weergave wanneer het
maintainergeheim aanwezig is.

## Canonieke scenariodekking

Het `taxonomy.yaml`-bestand in de hoofdmap definieert semantische
dekkings-id's. Scenario-YAML-bestanden onder `qa/scenarios/` koppelen elk
scenario aan die id's en beheren de uitvoeringsmetadata:
`channel` is de enige kanaalvereiste en `profiles` declareert
lidmaatschap van benoemde uitvoeringen. Het kanaalstuurprogramma is een
uitwisselbare implementatiekeuze op uitvoeringsniveau. TypeScript-runners
bevragen die catalogus; ze onderhouden geen parallelle inventarissen van
scenario's of dekking.

Statische uitvoer van `qa coverage` rapporteert de koppeling tussen de
taxonomie en scenario's. Het daadwerkelijke bewijs komt van
`qa-evidence.json`, dat het uitgevoerde scenario, dekkings-id's, het kanaal, het
daadwerkelijk gebruikte stuurprogramma en het resultaat vastlegt. Kanaal en
stuurprogramma zijn rapportdimensies, geen aanvullende vocabulaire voor
dekkings-id's of assen voor scenariogeschiktheid.

Voer voor een tijdelijke Linux-VM-lane zonder Docker in het QA-pad te
introduceren het volgende uit:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Hiermee wordt een nieuwe Multipass-gast opgestart, worden afhankelijkheden
geïnstalleerd, wordt OpenClaw binnen de gast gebouwd, wordt
`qa suite` uitgevoerd en worden vervolgens het normale QA-rapport en de
samenvatting teruggekopieerd naar `.artifacts/qa-e2e/...` op de host. Dit gebruikt
hetzelfde gedrag voor scenarioselectie als `qa suite` op de host.

Suite-uitvoeringen op de host en in Multipass voeren standaard meerdere
geselecteerde scenario's parallel uit met geïsoleerde Gateway-workers.
`qa-channel` gebruikt standaard concurrency 4, begrensd door het aantal
geselecteerde scenario's. Gebruik `--concurrency
<count>` om het aantal workers af
te stemmen of `--concurrency 1` voor seriële uitvoering. Gebruik
`--pack personal-agent` om het benchmarkpakket voor de persoonlijke assistent (10
scenario's) uit te voeren. De pakketselector is additief met herhaalde
`--scenario`-vlaggen: expliciete scenario's worden eerst uitgevoerd,
daarna worden pakketscenario's in pakketvolgorde uitgevoerd waarbij duplicaten
worden verwijderd. Gebruik `--pack observability` om de scenario's
`otel-trace-smoke` en `docker-prometheus-smoke` samen te selecteren wanneer een
aangepaste QA-runner de configuratie van de OpenTelemetry-collector al levert.

De opdracht sluit af met een niet-nulcode wanneer een scenario mislukt. Gebruik
`--allow-failures` wanneer je artifacts wilt zonder een mislukte afsluitcode.

Live-uitvoeringen sturen de ondersteunde QA-authenticatie-invoer door die
praktisch is voor de gast: providerkeys uit omgevingsvariabelen, het pad naar
de configuratie van de live QA-provider en `CODEX_HOME` wanneer deze
aanwezig is. Houd `--output-dir` onder de hoofdmap van de repository, zodat
de gast via de gekoppelde werkruimte kan terugschrijven.

## QA-naslag voor Discord, Slack, Telegram en WhatsApp

De Matrix-adapter gebruikt de hierboven gedocumenteerde tijdelijke,
door Docker ondersteunde lane. Discord, Slack, Telegram en WhatsApp werken met
vooraf bestaande echte transports, dus de naslaginformatie daarvoor staat
hier.

### Gedeelde CLI-vlaggen

Deze lanes worden geregistreerd via
`extensions/qa-lab/src/live-transports/shared/live-transport-cli.ts` en accepteren dezelfde vlaggen:

| Vlag                                  | Standaardwaarde                                    | Beschrijving                                                                                                                                     |
| ------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `--scenario <id>`                     | -                                                  | Voer alleen dit scenario uit. Herhaalbaar.                                                                                                      |
| `--output-dir <path>`                 | `<repo>/.artifacts/qa-e2e/<transport>-<timestamp>` | Hier worden rapporten, samenvattingen, bewijs, transportspecifieke artifacts en het uitvoerlogboek geschreven. Relatieve paden worden ten opzichte van `--repo-root` bepaald. |
| `--repo-root <path>`                  | `process.cwd()`                                    | Hoofdmap van de repository bij aanroepen vanuit een neutrale huidige werkmap.                                                                   |
| `--sut-account <id>`                  | `sut`                                              | Tijdelijke account-id binnen de configuratie van de QA-gateway.                                                                                 |
| `--provider-mode <mode>`              | `live-frontier`                                    | `mock-openai`, `aimock` of `live-frontier`.                                                                                                     |
| `--model <ref>` / `--alt-model <ref>` | standaardwaarde van provider                        | Primaire/alternatieve modelrefs.                                                                                                                |
| `--fast`                              | uit                                                | Snelle providermodus waar ondersteund.                                                                                                          |
| `--credential-source <env\|convex>`   | `env`                                              | Zie [Pool met Convex-aanmeldgegevens](#convex-credential-pool).                                                                                 |
| `--credential-role <maintainer\|ci>`  | `ci` in CI, anders `maintainer`                 | Rol die wordt gebruikt wanneer `--credential-source convex`.                                                                                              |
| `--allow-failures`                    | uit                                                | Schrijf artifacts zonder een mislukte afsluitcode te retourneren wanneer scenario's mislukken.                                                  |

Elke lane sluit af met een niet-nulcode als een scenario mislukt.
`--allow-failures` schrijft artifacts zonder een mislukte afsluitcode in te
stellen. Telegram accepteert ook `--list-scenarios` om beschikbare
scenario-id's af te drukken en af te sluiten; de andere lanes bieden die vlag
niet aan.

### Telegram-QA

```bash
pnpm openclaw qa telegram
```

Richt zich op één echte privé-Telegram-groep met twee afzonderlijke bots
(stuurprogramma + SUT). De SUT-bot moet een Telegram-gebruikersnaam hebben;
bot-naar-bot-waarneming werkt het beste wanneer voor beide bots
**Bot-to-Bot Communication Mode** is ingeschakeld in `@BotFather`.

Vereiste omgevingsvariabelen wanneer `--credential-source env`:

- `OPENCLAW_QA_TELEGRAM_GROUP_ID` - numerieke chat-id (tekenreeks).
- `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`

Het profiel `release` selecteert de onderhouden
Telegram-YAML-scenario's; `all` voegt optionele controles toe voor
sessies, gebruik, antwoordketens en streamingbelasting. Expliciete waarden voor
`--scenario` overschrijven het profiel.

- `channel-canary`
- `channel-mention-gating`
- `telegram-help-command`
- `telegram-commands-command`
- `telegram-tools-compact-command`
- `telegram-whoami-command`
- `telegram-status-command`
- `telegram-repeated-command-authorization`
- `telegram-other-bot-command-gating`
- `telegram-context-command`
- `telegram-current-session-status-tool`
- `telegram-tool-only-usage-footer`
- `telegram-reply-chain-exact-marker`
- `telegram-stream-final-single-message`
- `telegram-long-final-reuses-preview`
- `telegram-long-final-three-chunks`

Het profiel `release` omvat altijd canary, mention-gating, antwoorden op native opdrachten, opdrachtadressering en groepsantwoorden van bot naar bot. `mock-openai`
omvat ook de deterministische controle van de lange definitieve preview.
`telegram-current-session-status-tool` en
`telegram-tool-only-usage-footer` blijven opt-in: de eerste is alleen stabiel
wanneer die direct na canary in een thread wordt uitgevoerd, en de tweede is bewijs met echte Telegram
van de footer `/usage` bij antwoorden die alleen uit tools bestaan. Gebruik `pnpm openclaw qa telegram
--list-scenarios --provider-mode mock-openai` om de huidige
standaard/optionele verdeling met regressiereferenties af te drukken. Gebruik `--profile all` voor elk
Telegram-scenario met de live-adapter.

Uitvoerartefacten:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - bewijsvermeldingen voor de live-transportcontroles,
  inclusief velden voor profiel, dekking, provider, kanaal, artefacten, resultaat en RTT.

Telegram-runs van pakketten gebruiken hetzelfde contract voor Telegram-inloggegevens. Herhaalde RTT-
meting maakt deel uit van de normale live Telegram-lane voor pakketten; de RTT-
verdeling wordt voor de geselecteerde RTT-controle onder `result.timing` opgenomen in `qa-evidence.json`.

```bash
OPENCLAW_QA_CREDENTIAL_SOURCE=convex \
pnpm test:docker:npm-telegram-live
```

Wanneer `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` is ingesteld, leaset de live-wrapper van het pakket
een `kind: "telegram"`-inloggegeven, exporteert die de geleasete omgevingsvariabelen voor groep/driver/SUT-
bot naar de run van het geïnstalleerde pakket, stuurt die heartbeats voor de lease en geeft die
bij afsluiten vrij. De pakketwrapper gebruikt standaard 20 RTT-controles van
`channel-canary`, een RTT-time-out van 30s en buiten CI de Convex-rol
`maintainer` wanneer Convex is geselecteerd. Overschrijf
`OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`, `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`
of `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` om de RTT-meting af te stemmen zonder
een afzonderlijke RTT-opdracht of Telegram-specifieke samenvattingsindeling
te maken.

### Discord-QA

```bash
pnpm openclaw qa discord
```

Richt zich op één echt privé-Discord-guildkanaal met twee bots: een driverbot
die door het harnas wordt bestuurd en een SUT-bot die door de onderliggende OpenClaw-Gateway
via de gebundelde Discord-Plugin wordt gestart. Controleert de verwerking van kanaalvermeldingen, of
de SUT-bot de native opdracht `/help` bij Discord heeft geregistreerd, en
opt-in-bewijsscenario's voor Mantis.

Vereiste omgevingsvariabelen wanneer `--credential-source env`:

- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` - moet overeenkomen met de gebruikers-id van de SUT-bot
  die Discord retourneert (anders mislukt de lane onmiddellijk).

Optioneel:

- `OPENCLAW_QA_DISCORD_VOICE_CHANNEL_ID` selecteert het spraak-/podiumkanaal voor
  `discord-voice-autojoin`; zonder deze waarde selecteert het scenario het eerste zichtbare
  spraak-/podiumkanaal voor de SUT-bot.

Discord-scenario's van YAML-modules (`qa/scenarios/channels/discord-*.yaml`):

- `discord-canary`
- `discord-mention-gating`
- `discord-native-help-command-registration`
- `discord-voice-autojoin` - opt-in-spraakscenario. Wordt afzonderlijk uitgevoerd, schakelt
  `channels.discord.voice.autoJoin` in en controleert of de huidige
  Discord-spraakstatus van de SUT-bot het doelspraak-/podiumkanaal is. Convex-inloggegevens voor Discord
  kunnen optioneel `voiceChannelId` bevatten; anders detecteert de runner-
  adapter het eerste zichtbare spraak-/podiumkanaal in de guild.
- `discord-status-reactions-tool-only` - opt-in-scenario voor Mantis. Wordt
  afzonderlijk uitgevoerd omdat het de SUT omschakelt naar altijd ingeschakelde guildantwoorden
  die alleen uit tools bestaan met `messages.statusReactions.enabled=true`, en legt vervolgens een REST-
  reactietijdlijn plus visuele HTML-/PNG-artefacten vast. Mantis-rapporten van voor en na
  behouden ook door het scenario geleverde MP4-artefacten als `baseline.mp4`
  en `candidate.mp4`.
- `discord-thread-reply-filepath-attachment` - opt-in-scenario voor Mantis; zie
  [Discord-scenario's voor Mantis](#discord-mantis-scenarios).

Voer het Discord-scenario voor automatisch deelnemen aan spraak expliciet uit:

```bash
pnpm openclaw qa discord \
  --scenario discord-voice-autojoin \
  --provider-mode mock-openai
```

Voer het Mantis-scenario voor statusreacties expliciet uit:

```bash
pnpm openclaw qa discord \
  --scenario discord-status-reactions-tool-only \
  --provider-mode live-frontier \
  --model openai/gpt-5.6-luna \
  --alt-model openai/gpt-5.6-luna \
  --fast
```

Uitvoerartefacten:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - bewijsvermeldingen voor de live-transportcontroles.
- `discord-qa-reaction-timelines.json` en
  `discord-status-reactions-tool-only-timeline.png` wanneer het statusreactiescenario
  wordt uitgevoerd.

### Slack-QA

```bash
pnpm openclaw qa slack
```

Richt zich op één echt privé-Slack-kanaal met twee verschillende bots: een driverbot
die door het harnas wordt bestuurd en een SUT-bot die door de onderliggende OpenClaw-Gateway
via de gebundelde Slack-Plugin wordt gestart.

Vereiste omgevingsvariabelen wanneer `--credential-source env`:

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`

Optioneel:

- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` schakelt visuele goedkeurings-
  controlepunten voor Mantis in. De adapter schrijft `<scenario>.pending.json` en
  `<scenario>.resolved.json` en wacht vervolgens op overeenkomende `.ack.json`-bestanden.
- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_TIMEOUT_MS` overschrijft de time-out voor
  bevestiging van controlepunten. De standaardwaarde is `120000`.

Canonieke YAML-scenario's die via de live Slack-adapter beschikbaar zijn:

- `thread-follow-up`
- `thread-isolation`

Slack-scenario's van YAML-modules (`qa/scenarios/channels/slack-*.yaml`):

- `slack-canary`
- `slack-mention-gating`
- `slack-allowlist-block`
- `slack-channel-disabled-warning` - opt-in-probe met echte Slack die bevestigt dat een
  geconfigureerd uitgeschakeld kanaal een gestructureerde waarschuwing genereert zonder te antwoorden.
- `slack-top-level-reply-shape`
- `slack-restart-resume`
- `slack-progress-commentary-true`, `slack-progress-commentary-false`,
  `slack-progress-commentary-omitted` en
  `slack-progress-commentary-verbose-dedupe` - opt-in-probes met echte Slack voor
  onafhankelijke besturing van commentaar/toolvoortgang, de verouderde
  standaardwaarde bij een weggelaten sleutel en eenmalige aflevering wanneer duurzame uitgebreide voortgang is ingeschakeld.
- `slack-reaction-glyph-native` - opt-in-live-scenario voor reacties van de berichttool.
  Instrueert de agent om exact het teken `✅` door te geven en bevestigt dat Slack
  `white_check_mark` voor de SUT-bot bij het doelbericht heeft opgeslagen.
- `slack-chart-presentation-native` - opt-in-scenario voor draagbare grafieken dat
  het native blok `data_visualization` en de exacte toegankelijke tekst controleert.
- `slack-table-presentation-native` - opt-in-scenario voor draagbare tabellen dat
  het native blok `data_table`, de exacte rijen en de toegankelijke tekst controleert.
- `slack-table-invalid-blocks-fallback` - opt-in-scenario voor rechtstreeks transport
  dat een structureel leesbare onbewerkte tabel boven de limiet met 101 gegevensrijen
  plus de koptekst via het
  productiepad voor Slack-verzending verzendt, bewijst dat Slack zelf `invalid_blocks`
  retourneert en controleert of de opgeslagen terugval zonder opmaak volledig is en geen
  native gegevensblok bevat. Scenariodetails behouden alleen veilig bewijs voor foutcode, aantal en
  booleaanse waarden.
- `slack-approval-exec-native` - opt-in-scenario voor native Slack-goedkeuring van exec.
  Vraagt via de Gateway om goedkeuring van exec, controleert of het Slack-bericht
  native goedkeuringsknoppen bevat, verwerkt de goedkeuring en controleert de bijgewerkte Slack-status
  na verwerking.
- `slack-approval-plugin-native` - opt-in-scenario voor native Slack-Plugin-goedkeuring.
  Schakelt het doorsturen van exec- en Plugin-goedkeuringen samen in, zodat Plugin-
  gebeurtenissen niet worden onderdrukt door routering van exec-goedkeuringen, en controleert vervolgens hetzelfde
  native Slack-UI-pad voor in behandeling/verwerkt.
- `slack-codex-approval-exec-native` - opt-in-scenario voor opdrachtgoedkeuring door Codex Guardian.
  Schakelt de Codex-Plugin in Guardian-modus in, routeert een
  door Slack geïnitieerde Gateway-agentbeurt via het Codex-app-serverharnas,
  wacht op de native Slack-prompt voor Plugin-goedkeuring voor
  `openclaw-codex-app-server`, verwerkt deze en controleert of de Codex-beurt
  eindigt met de verwachte markeringen voor opdrachtuitvoer en assistent.
- `slack-codex-approval-plugin-native` - opt-in-scenario voor bestandsgoedkeuring door Codex Guardian.
  Gebruikt een instructie `apply_patch` buiten de werkruimte, zodat Codex
  de app-serverroute voor goedkeuring van bestandswijzigingen activeert, en controleert vervolgens hetzelfde native
  Slack-goedkeuringspad voor in behandeling/verwerkt, de definitieve assistentmarkering en de exacte bestands-
  inhoud vóór het opruimen.

De Codex-goedkeuringsscenario's vereisen een `openai/*` of `codex/*` `--model`, de
normale inloggegevens voor het live model en Codex-authenticatie of API-sleutelauthenticatie die door de Codex-Plugin wordt geaccepteerd.
De scenariodetails omvatten de Codex-app-servermethode, de geselecteerde Codex-model-
sleutel, de definitieve status van de Codex-beurt en controle van de bewerkingsmarkering naast de
geredigeerde Slack-goedkeuringsmetadata.

Uitvoerartefacten:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - bewijsvermeldingen voor de live-transportcontroles.
- `approval-checkpoints/` - alleen wanneer Mantis
  `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` instelt; bevat JSON voor controlepunten,
  JSON voor bevestigingen en schermafbeeldingen van in behandeling/verwerkt.

#### De Slack-werkruimte instellen

De lane heeft twee verschillende Slack-apps in één werkruimte nodig, plus een kanaal waarvan beide
bots lid zijn:

- `channelId` - de `Cxxxxxxxxxx`-id van een kanaal waarvoor beide bots zijn
  uitgenodigd. Gebruik een speciaal hiervoor bestemd kanaal; de lane plaatst bij elke run berichten.
- `driverBotToken` - bottoken (`xoxb-...`) van de **Driver**-app.
- `sutBotToken` - bottoken (`xoxb-...`) van de **SUT**-app, die een
  andere Slack-app dan de driver moet zijn, zodat de gebruikers-id van de bot verschilt.
- `sutAppToken` - token op appniveau (`xapp-...`) van de SUT-app met
  `connections:write`, gebruikt door Socket Mode zodat de SUT-app gebeurtenissen kan ontvangen.

Gebruik bij voorkeur een Slack-werkruimte die specifiek voor QA bestemd is, in plaats van een productie-
werkruimte opnieuw te gebruiken.

Het onderstaande SUT-manifest beperkt bewust de productie-installatie van de
gebundelde Slack-Plugin (`extensions/slack/src/setup-shared.ts:12`) tot de
machtigingen en gebeurtenissen die door de live Slack-QA-suite worden gedekt. Raadpleeg voor de
configuratie van het productiekanaal zoals gebruikers die zien
[Snelle configuratie van het Slack-kanaal](/nl/channels/slack#quick-setup); het QA-paar Driver/SUT
is bewust afzonderlijk omdat de lane twee verschillende botgebruikers-
id's in één werkruimte nodig heeft.

**1. De Driver-app maken**

Ga naar [api.slack.com/apps](https://api.slack.com/apps) → _Create New App_ →
_From a manifest_ → kies de QA-werkruimte, plak het volgende manifest
en kies vervolgens _Install to Workspace_:

```json
{
  "display_information": {
    "name": "OpenClaw QA Driver",
    "description": "Testdriverbot voor de live Slack-lane van OpenClaw QA"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA Driver",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": ["chat:write", "channels:history", "groups:history", "users:read"]
    }
  },
  "settings": {
    "socket_mode_enabled": false
  }
}
```

Kopieer de _Bot User OAuth Token_ (`xoxb-...`) - dit wordt
`driverBotToken`. De driver hoeft alleen berichten te plaatsen en zichzelf te identificeren;
geen gebeurtenissen, geen Socket Mode.

**2. De SUT-app maken**

Herhaal _Create New App → From a manifest_ in dezelfde werkruimte. Deze QA-app
gebruikt bewust een beperktere versie van het productie-
manifest van de gebundelde Slack-Plugin (`extensions/slack/src/setup-shared.ts:12`): reactie-
bereiken en -gebeurtenissen zijn weggelaten omdat de live Slack-QA-suite
de verwerking van reacties nog niet dekt.

```json
{
  "display_information": {
    "name": "OpenClaw QA SUT",
    "description": "OpenClaw QA SUT connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA SUT",
      "always_online": true
    },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

Nadat Slack de app heeft aangemaakt, doe je twee dingen op de instellingenpagina:

- _Install to Workspace_ → kopieer de _Bot User OAuth Token_ → die wordt
  `sutBotToken`.
- _Basic Information → App-Level Tokens → Generate Token and Scopes_ → voeg
  scope `connections:write` toe → sla op → kopieer de waarde `xapp-...` → die
  wordt `sutAppToken`.

Controleer of de twee bots verschillende gebruikers-id's hebben door voor elk
token `auth.test` aan te roepen. De runtime onderscheidt de driver en SUT op basis van gebruikers-id; als je één app
voor beide hergebruikt, mislukt de vermeldingsfiltering onmiddellijk.

**3. Maak het kanaal**

Maak in de QA-workspace een kanaal (bijvoorbeeld `#openclaw-qa`) en nodig beide
bots vanuit het kanaal uit:

```text
/invite @OpenClaw QA Driver
/invite @OpenClaw QA SUT
```

Kopieer de id `Cxxxxxxxxxx` uit _channel info → About → Channel ID_ - die
wordt `channelId`. Een openbaar kanaal werkt; als je een privékanaal gebruikt,
hebben beide apps al `groups:history`, zodat het lezen van de geschiedenis door de harness
nog steeds slaagt.

**4. Registreer de inloggegevens**

Er zijn twee opties. Gebruik omgevingsvariabelen voor foutopsporing op één machine (stel de vier
`OPENCLAW_QA_SLACK_*`-variabelen in en geef `--credential-source env` door), of vul
de gedeelde Convex-pool, zodat CI en andere beheerders deze kunnen leasen.

Schrijf voor de Convex-pool de vier velden naar een JSON-bestand:

```json
{
  "channelId": "Cxxxxxxxxxx",
  "driverBotToken": "xoxb-...",
  "sutBotToken": "xoxb-...",
  "sutAppToken": "xapp-..."
}
```

Met `OPENCLAW_QA_CONVEX_SITE_URL` en `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
geëxporteerd in je shell registreer en controleer je:

```bash
pnpm openclaw qa credentials add \
  --kind slack \
  --payload-file slack-creds.json \
  --note "QA Slack pool seed"

pnpm openclaw qa credentials list --kind slack --status all --json
```

Verwacht `count: 1`, `status: "active"` en geen veld `lease`.

**5. Controleer end-to-end**

Voer de lane lokaal uit om te bevestigen dat beide bots via de
broker met elkaar kunnen communiceren:

```bash
pnpm openclaw qa slack \
  --credential-source convex \
  --credential-role maintainer \
  --output-dir .artifacts/qa-e2e/slack-local
```

Een geslaagde run is ruim binnen 30 seconden voltooid en `qa-suite-report.md`
toont zowel `slack-canary` als `slack-mention-gating` met status `pass`. Als de
lane ongeveer 90 seconden blijft hangen en afsluit met `Convex credential pool exhausted
for kind "slack"`, is de pool leeg of is elke rij geleaset - `qa
credentials list --kind slack --status all --json` geeft aan welke situatie van toepassing is.

### WhatsApp-QA

```bash
pnpm openclaw qa whatsapp
```

Richt zich op twee speciale WhatsApp Web-accounts: een driveraccount dat wordt bestuurd door
de harness en een SUT-account dat door de onderliggende OpenClaw-Gateway wordt gestart via
de gebundelde WhatsApp-Plugin.

Vereiste omgevingsvariabelen wanneer `--credential-source env`:

- `OPENCLAW_QA_WHATSAPP_DRIVER_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_SUT_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_DRIVER_AUTH_ARCHIVE_BASE64`
- `OPENCLAW_QA_WHATSAPP_SUT_AUTH_ARCHIVE_BASE64`

Optioneel:

- `OPENCLAW_QA_WHATSAPP_GROUP_JID` schakelt groepsscenario's in, zoals
  `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-broadcast-group-fanout`, `whatsapp-group-activation-always`,
  `whatsapp-group-reply-to-bot-triggers`, scenario's voor groepsacties, media en peilingen,
  en `whatsapp-group-allowlist-block`.

WhatsApp-YAML-scenario's (`qa/scenarios/channels/whatsapp-*.yaml`):

- Basislijn en groepsfiltering: `whatsapp-canary`, `whatsapp-pairing-block`,
  `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-group-activation-always`, `whatsapp-group-reply-to-bot-triggers`,
  `whatsapp-top-level-reply-shape`, `whatsapp-restart-resume`,
  `whatsapp-group-allowlist-block`.
- Native opdrachten: `whatsapp-help-command`, `whatsapp-status-command`,
  `whatsapp-commands-command`, `whatsapp-tools-compact-command`,
  `whatsapp-whoami-command`, `whatsapp-context-command`,
  `whatsapp-native-new-command`.
- Gedrag voor antwoorden en definitieve uitvoer: `whatsapp-tool-only-usage-footer`,
  `whatsapp-reply-to-message`, `whatsapp-group-reply-to-message`,
  `whatsapp-reply-to-mode-batched`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`, `whatsapp-stream-final-message-accounting`.
- Berichtacties via het gebruikerstraject: `whatsapp-agent-message-action-react` begint
  met een echt privébericht van de driver, laat het model de tool `message` aanroepen en
  observeert de native WhatsApp-reactie. `whatsapp-agent-message-action-upload-file`
  gebruikt dezelfde opzet voor `message(action=upload-file)` en observeert
  native WhatsApp-media. `whatsapp-group-agent-message-action-react` en
  `whatsapp-group-agent-message-action-upload-file` bewijzen dezelfde
  voor gebruikers zichtbare acties in een echte WhatsApp-groep.
- Groepsfan-out: `whatsapp-broadcast-group-fanout` begint met één WhatsApp-groepsbericht
  met een vermelding en controleert afzonderlijke zichtbare antwoorden van `main`
  en `qa-second`.
- Groepsactivering: `whatsapp-group-activation-always` wijzigt een echte
  groepssessie naar `/activation always`, bewijst dat een groepsbericht zonder vermelding
  de agent activeert en herstelt vervolgens `/activation mention`.
  `whatsapp-group-reply-to-bot-triggers` plaatst eerst een botantwoord, stuurt daarop een native
  geciteerd antwoord zonder expliciete vermelding en controleert of de agent
  vanuit die antwoordcontext wordt geactiveerd.
- Inkomende media en gestructureerde berichten: `whatsapp-inbound-image-caption`,
  `whatsapp-audio-preflight`, `whatsapp-inbound-structured-messages`,
  `whatsapp-group-audio-gating`, `whatsapp-inbound-reaction-no-trigger`.
  Deze sturen echte WhatsApp-afbeeldings-, audio-, document-, locatie-, contact-,
  sticker- en reactiegebeurtenissen via de driver.
- Directe Gateway-contractprobes: `whatsapp-outbound-media-matrix`,
  `whatsapp-outbound-document-preserves-filename`, `whatsapp-outbound-poll`,
  `whatsapp-outbound-send-serialization`,
  `whatsapp-group-outbound-media`, `whatsapp-group-outbound-poll`,
  `whatsapp-message-actions`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`. Deze omzeilen bewust modelprompts
  en bewijzen deterministische contracten voor Gateway/kanaal-`send`, `poll` en
  `message.action`.
- Dekking van toegangsbeheer: `whatsapp-access-control-dm-open`,
  `whatsapp-access-control-dm-disabled`, `whatsapp-access-control-group-open`,
  `whatsapp-access-control-group-disabled`, `whatsapp-group-allowlist-block`.
- Native goedkeuringen: `whatsapp-approval-exec-deny-native`,
  `whatsapp-approval-exec-native`, `whatsapp-approval-exec-reaction-native`,
  `whatsapp-approval-exec-group-reaction-native`,
  `whatsapp-approval-plugin-native`.
- Statusreacties: `whatsapp-status-reactions`,
  `whatsapp-status-reaction-lifecycle`.

De catalogus bevat momenteel 52 scenario's. De standaardlane `live-frontier`
blijft met 8 scenario's klein voor snelle rooktestdekking. De standaardlane `mock-openai`
voert 39 scenario's deterministisch uit via het echte WhatsApp-
transport, waarbij alleen de modeluitvoer wordt gemockt; goedkeuringsscenario's en enkele
zwaardere/blokkerende controles blijven expliciet op scenario-id geselecteerd.

De WhatsApp-QA-driver observeert gestructureerde livegebeurtenissen (`text`, `media`,
`location`, `reaction` en `poll`) en kan actief media, peilingen,
contacten, locaties en stickers verzenden. QA Lab importeert die driver via het
pakketoppervlak `@openclaw/whatsapp/api.js` in plaats van private
WhatsApp-runtimebestanden rechtstreeks te benaderen. Voor groepsobservaties is `fromJid` de groeps-JID,
terwijl `participantJid` en `fromPhoneE164` de verzendende deelnemer identificeren.
Berichtinhoud wordt standaard geredigeerd. Directe Gateway-probes voor peilingen, bestandsuploads,
media, groepspeilingen, groepsmedia en antwoordvormen zijn controles van transport-/API-
contracten; ze gelden niet als bewijs dat een gebruikersprompt de
agent dezelfde actie liet kiezen. Bewijs van acties via het gebruikerstraject komt uit scenario's
zoals `whatsapp-agent-message-action-react` en
`whatsapp-group-agent-message-action-react`, waarin de driver een normaal
WhatsApp-bericht verzendt en QA Lab het resulterende native WhatsApp-artefact observeert.
Details van WhatsApp-scenario's bevatten de opzet van elk scenario (`user-path`,
`direct-gateway` of `native-approval`), zodat bewijs niet kan worden aangezien voor een
sterker contract dan het daadwerkelijk aantoont.

Uitvoerartefacten:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - bewijsitems voor de live transportcontroles.

### Convex-pool voor inloggegevens

Discord-, Slack-, Telegram- en WhatsApp-lanes kunnen inloggegevens leasen uit een
gedeelde Convex-pool in plaats van de bovenstaande omgevingsvariabelen te lezen. Geef
`--credential-source convex` door (of stel `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` in);
QA Lab verkrijgt een exclusieve lease, stuurt gedurende de
run Heartbeats voor die lease en geeft deze bij het afsluiten vrij. Pooltypen zijn `"discord"`, `"slack"`,
`"telegram"` en `"whatsapp"`.

Payloadvormen die de broker valideert bij `admin/add`:

- Discord (`kind: "discord"`): `{ guildId: string, channelId: string,
driverBotToken: string, sutBotToken: string, sutApplicationId: string }`.
- Telegram (`kind: "telegram"`): `{ groupId: string, driverToken: string,
sutToken: string }` - `groupId` moet een numerieke chat-id-tekenreeks zijn.
- Echte Telegram-gebruiker (`kind: "telegram-user"`): `{ groupId: string, sutToken:
string, testerUserId: string, testerUsername: string, telegramApiId:
string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string,
tdlibArchiveBase64: string, tdlibArchiveSha256: string,
desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }` -
  uitsluitend voor Mantis-bewijs met Telegram Desktop. Algemene QA Lab-lanes mogen
  dit type niet verkrijgen.
- WhatsApp (`kind: "whatsapp"`): `{ driverPhoneE164: string, sutPhoneE164:
string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string,
groupJid?: string }` - telefoonnummers moeten verschillende E.164-tekenreeksen zijn.

De workflow voor Mantis-bewijs met Telegram Desktop houdt één exclusieve Convex-
lease `telegram-user` vast voor zowel de TDLib-CLI-driver als de Telegram Desktop-
getuige en geeft deze vrij nadat het bewijs is gepubliceerd.

Wanneer een pull request een deterministische visuele diff vereist, kan Mantis hetzelfde gemockte
modelantwoord gebruiken op `main` en op de head van de pull request terwijl de Telegram-formatter of
leveringslaag verandert. De standaardinstellingen voor opnamen zijn afgestemd op opmerkingen bij pull requests: standaard
Crabbox-klasse, bureaubladopname met 24fps, bewegings-GIF met 24fps en een voorbeeldbreedte
van 1920px. Opmerkingen met voor/na-vergelijkingen moeten een nette bundel publiceren die
alleen de bedoelde GIF's bevat.

Slack-lanes kunnen de pool ook gebruiken. Controles van de Slack-payloadvorm bevinden zich momenteel
in de Slack-QA-runner in plaats van in de broker; gebruik `{ channelId: string,
driverBotToken: string, sutBotToken: string, sutAppToken: string }`, met een
Slack-kanaal-id zoals `Cxxxxxxxxxx`. Zie
[De Slack-workspace instellen](#setting-up-the-slack-workspace) voor het inrichten van apps
en scopes.

Operationele omgevingsvariabelen en het contract voor het Convex-brokereindpunt staan in
[Testen → Gedeelde Telegram-inloggegevens via Convex](/nl/help/testing#shared-telegram-credentials-via-convex-v1)
(de sectienaam dateert van vóór de pool met meerdere kanalen; de leasesemantiek wordt
door alle typen gedeeld).

## Seeds uit de repository

Seed-assets staan in `qa/`:

- `qa/scenarios/index.yaml`
- `qa/scenarios/<theme>/*.yaml`

Deze staan bewust in git, zodat het QA-plan zichtbaar is voor zowel mensen als
de agent.

`qa-lab` blijft een algemene YAML-scenariorunner. Elk YAML-scenariobestand is de
bron van waarheid voor één testrun en moet het volgende definiëren:

- `title` op het hoogste niveau
- metadata voor `scenario`
- optionele metadata voor categorie, capability, lane en risico in `scenario`
- documentatie- en codereferenties in `scenario`
- optionele Plugin-vereisten in `scenario`
- optionele patch voor Gateway-configuratie in `scenario`
- uitvoerbare `flow` op het hoogste niveau voor flow-scenario's, of
  `scenario.execution.kind` / `scenario.execution.path` voor Vitest- en
  Playwright-scenario's

Het herbruikbare runtime-oppervlak waarop `flow` steunt, blijft generiek en
domeinoverstijgend. YAML-scenario's kunnen bijvoorbeeld helpers aan de transportzijde
combineren met helpers aan de browserzijde die de ingebedde Control UI via
de Gateway-`browser.request`-naad aansturen, zonder een runner voor een speciaal geval toe te voegen.

Scenariobestanden moeten worden gegroepeerd op productcapaciteit in plaats van op map
in de bronstructuur. Houd scenario-ID's stabiel wanneer bestanden worden verplaatst; gebruik `docsRefs` en
`codeRefs` voor traceerbaarheid van de implementatie.

De basislijst moet breed genoeg blijven om het volgende te dekken:

- DM- en kanaalchat
- threadgedrag
- levenscyclus van berichtacties
- cron-callbacks
- geheugenoproep
- wisselen van model
- overdracht aan subagent
- lezen van repo en documentatie
- één kleine bouwtaak, zoals Lobster Invaders

## Mocklanen voor providers

`qa suite` heeft twee lokale mocklanen voor providers:

- `mock-openai` is de scenariobewuste OpenClaw-mock. Dit blijft de standaard
  deterministische mocklaan voor QA op basis van de repo en pariteitsgates.
- `aimock` start een provider-server op basis van AIMock voor experimentele
  protocol-, fixture-, opname-/afspeel- en chaostests. Deze is aanvullend en
  vervangt de `mock-openai`-scenariodispatcher niet.

De implementatie van providerlanen bevindt zich onder `extensions/qa-lab/src/providers/`.
Elke provider beheert zijn standaardwaarden, het starten van de lokale server, de Gateway-modelconfiguratie,
de vereisten voor het klaarzetten van auth-profielen en de capaciteitsvlaggen voor live/mock. Gedeelde suite- en
Gateway-code routeert via het providerregister in plaats van te vertakken op
providernamen.

## Transportadapters

`qa-lab` beheert een generieke transportnaad voor YAML-QA-scenario's. `qa-channel` is
de synthetische standaard. `crabline` start lokale servers in de vorm van providers en
voert de normale kanaalplugins van OpenClaw daarop uit. `live` is gereserveerd voor
echte providerreferenties en externe kanalen.

Op architectuurniveau is de verdeling:

- `qa-lab` beheert generieke scenario-uitvoering, workerconcurrency, het schrijven
  van artefacten en rapportage.
- De transportadapter beheert Gateway-configuratie, gereedheid, observatie van inkomend en uitgaand
  verkeer, transportacties en genormaliseerde transportstatus.
- YAML-scenariobestanden onder `qa/scenarios/` definiëren de testrun; `qa-lab`
  biedt het herbruikbare runtime-oppervlak dat ze uitvoert.

### Een kanaal toevoegen

Het toevoegen van een kanaal aan het YAML-QA-systeem vereist de kanaalimplementatie
plus een scenariopakket dat het kanaalcontract beproeft. Voeg voor smoke-CI-
dekking de bijbehorende lokale Crabline-provider-server toe en stel deze beschikbaar
via het `crabline`-stuurprogramma.

Voeg geen nieuwe QA-opdrachthoofdstructuur op het hoogste niveau toe wanneer de gedeelde `qa-lab`-host
de flow kan beheren.

`qa-lab` beheert de gedeelde hostmechanismen:

- de `openclaw qa`-opdrachthoofdstructuur
- opstarten en afsluiten van de suite
- workerconcurrency
- schrijven van artefacten
- genereren van rapporten
- uitvoeren van scenario's
- compatibiliteitsaliassen voor oudere `qa-channel`-scenario's

Runnerplugins beheren het transportcontract:

- hoe `openclaw qa <runner>` onder de gedeelde `qa`-hoofdstructuur wordt gekoppeld
- hoe de Gateway voor dat transport wordt geconfigureerd
- hoe gereedheid wordt gecontroleerd
- hoe inkomende gebeurtenissen worden geïnjecteerd
- hoe uitgaande berichten worden geobserveerd
- hoe transcripties en genormaliseerde transportstatus beschikbaar worden gesteld
- hoe door transport ondersteunde acties worden uitgevoerd
- hoe transportspecifieke reset of opschoning wordt afgehandeld

De minimale adoptiedrempel voor een nieuw kanaal:

1. Behoud `qa-lab` als beheerder van de gedeelde `qa`-hoofdstructuur.
2. Implementeer de transportrunner op de gedeelde `qa-lab`-hostnaad.
3. Houd transportspecifieke mechanismen binnen de runnerplugin of het kanaal-
   harnas.
4. Koppel de runner als `openclaw qa <runner>` in plaats van een
   concurrerende hoofdopdracht te registreren. Runnerplugins moeten `qaRunners` declareren in
   `openclaw.plugin.json` en een overeenkomende `qaRunnerCliRegistrations`-
   array exporteren vanuit `runtime-api.ts`. Houd `runtime-api.ts` lichtgewicht; luie CLI- en
   runneruitvoering moeten achter afzonderlijke toegangspunten blijven. Een optionele
   `adapterFactory` stelt het transport beschikbaar aan gedeelde scenario's zonder
   de bestaande scenariocatalogus van de opdracht te wijzigen. Partities van hetzelfde kanaal zijn serieel,
   tenzij de factory declareert dat elke instantie over geïsoleerde referenties of
   wegwerpservers, Gateway-status en artefactpaden beschikt.
5. Schrijf YAML-scenario's of pas ze aan onder de thematische `qa/scenarios/`-
   mappen.
6. Gebruik de generieke scenariohelpers voor nieuwe scenario's.
7. Houd bestaande compatibiliteitsaliassen werkend, tenzij de repo een
   opzettelijke migratie uitvoert.

De beslisregel is strikt:

- Als gedrag eenmaal in `qa-lab` kan worden uitgedrukt, plaats het dan in `qa-lab`.
- Als gedrag afhankelijk is van één kanaaltransport, houd het dan in die runner-
  plugin of dat pluginharnas.
- Als een scenario een nieuwe capaciteit nodig heeft die meer dan één kanaal kan gebruiken,
  voeg dan een generieke helper toe in plaats van een kanaalspecifieke vertakking in `suite.ts`.
- Als gedrag alleen betekenisvol is voor één transport, houd het scenario dan
  transportspecifiek en maak dat expliciet in het scenariocontract.

### Namen van scenariohelpers

Voorkeurshelpers voor nieuwe scenario's:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Compatibiliteitsaliassen blijven beschikbaar voor bestaande scenario's -
`waitForQaChannelReady`, `waitForOutboundMessage`, `waitForNoOutbound`,
`formatConversationTranscript`, `resetBus` - maar voor het schrijven van nieuwe scenario's
moeten de generieke namen worden gebruikt. De aliassen bestaan om een alles-in-één-keer-
migratie te voorkomen, niet als het toekomstige model.

## Rapportage

`qa-lab` exporteert een Markdown-protocolrapport uit de geobserveerde bustijdlijn.
Het rapport moet antwoord geven op:

- Wat werkte
- Wat mislukte
- Wat geblokkeerd bleef
- Welke vervolgsccenario's het toevoegen waard zijn

Voer voor de inventaris van beschikbare scenario's - nuttig bij het inschatten van vervolgwerk
of het aansluiten van een nieuw transport - `pnpm openclaw qa coverage` uit (voeg `--json` toe
voor machineleesbare uitvoer). Voer `pnpm openclaw qa coverage --match <query>` uit wanneer je gericht bewijs kiest voor aangeraakt
gedrag of een aangeraakt bestandspad. Het
overeenkomstenrapport doorzoekt scenariometagegevens, documentatiereferenties, codereferenties, dekkings-ID's,
plugins en providervereisten, en drukt vervolgens overeenkomende `qa suite
--scenario ...`-doelen af.

Elke `qa suite`-run schrijft `qa-evidence.json`-,
`qa-suite-summary.json`- en `qa-suite-report.md`-artefacten op het hoogste niveau voor de geselecteerde
scenarioset. Scenario's die `execution.kind: vitest` of
`execution.kind: playwright` declareren, voeren het overeenkomende testpad uit en schrijven ook
logs per scenario. Scenario's die `execution.kind: script` declareren, voeren de
bewijsproducent op `execution.path` uit via `node --import tsx` (waarbij
`${outputDir}` en `${scenarioId}` worden uitgebreid in `execution.args`); de
producent schrijft zijn eigen `qa-evidence.json`, waarvan de vermeldingen worden geïmporteerd in
de suite-uitvoer en waarvan de artefactpaden relatief ten opzichte van die
producent-`qa-evidence.json` worden opgelost. Wanneer `qa suite` via `qa run
--qa-profile` wordt bereikt, bevat dezelfde `qa-evidence.json` ook de
scorecardsamenvatting van het profiel voor de geselecteerde taxonomiecategorieën.

Behandel dekkingsuitvoer als hulpmiddel bij ontdekking, niet als vervanging van een gate; het
geselecteerde scenario heeft nog steeds de juiste providermodus, live transport,
Multipass, Testbox of releaselaan nodig voor het geteste gedrag. Zie voor
scorecardcontext [Volwassenheidsscorecard](/nl/maturity/scorecard).

Voer voor controles van karakter en stijl hetzelfde scenario uit voor meerdere live
modelreferenties en schrijf een beoordeeld Markdown-rapport:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.6-luna,thinking=medium,fast \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-8,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.6-sol,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-8,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

De opdracht voert lokale onderliggende QA-Gateway-processen uit, niet Docker. Scenario's voor karakter-
evaluatie moeten de persona instellen via `SOUL.md` en vervolgens gewone
gebruikersbeurten uitvoeren, zoals chat, werkruimtehulp en kleine bestandstaken. Het kandidaat-
model mag niet te horen krijgen dat het wordt geëvalueerd. De opdracht bewaart
elk volledig transcript, registreert basisstatistieken van de run en vraagt vervolgens de beoordelingsmodellen in
snelle modus met `xhigh`-redenering, waar ondersteund, om de runs te rangschikken op
natuurlijkheid, sfeer en humor. Gebruik `--blind-judge-models` bij het vergelijken van
providers: de beoordelingsprompt krijgt nog steeds elk transcript en elke runstatus, maar
kandidaatreferenties worden vervangen door neutrale labels zoals `candidate-01`; het
rapport koppelt rangschikkingen na het parsen terug aan echte referenties.

Kandidaatruns gebruiken standaard `high`-redenering, met `medium` voor GPT-5.6 Luna en
`xhigh` voor oudere OpenAI-evaluatiereferenties die dit ondersteunen. Overschrijf een specifieke
kandidaat inline met `--model provider/model,thinking=<level>`; inline-
opties ondersteunen ook `fast`, `no-fast` en `fast=<bool>`. `--thinking
<level>` stelt nog steeds een globale terugvalwaarde in en de oudere `--model-thinking
<provider/model=level>`-vorm blijft behouden voor compatibiliteit. OpenAI-kandidaat-
referenties gebruiken standaard de snelle modus, zodat prioriteitsverwerking wordt gebruikt waar de provider
dit ondersteunt. Geef `--fast` alleen door wanneer je de snelle modus voor
elk kandidaatmodel wilt afdwingen. De duur van kandidaten en beoordelaars wordt voor benchmarkanalyse
in het rapport vastgelegd, maar beoordelingsprompts zeggen expliciet dat niet op
snelheid mag worden gerangschikt. Runs van kandidaat- en beoordelingsmodellen gebruiken beide standaard concurrency 16.
Verlaag `--concurrency` of `--judge-concurrency` wanneer providerlimieten of lokale
Gateway-belasting een run te veel ruis geven.

Wanneer geen kandidaat-`--model` wordt doorgegeven, gebruikt de karakterevaluatie standaard
`openai/gpt-5.6-luna`, `openai/gpt-5.2`, `openai/gpt-5`,
`anthropic/claude-opus-4-8`, `anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` en `google/gemini-3.1-pro-preview`. Wanneer geen
`--judge-model` wordt doorgegeven, zijn de standaardbeoordelaars
`openai/gpt-5.6-sol,thinking=xhigh,fast` en
`anthropic/claude-opus-4-8,thinking=high`.

## Gerelateerde documentatie

- [Volwassenheidsscorecard](/nl/maturity/scorecard)
- [Benchmarkpakket voor persoonlijke agenten](/nl/concepts/personal-agent-benchmark-pack)
- [QA-kanaal](/nl/channels/qa-channel)
- [Testen](/nl/help/testing)
- [Dashboard](/nl/web/dashboard)
