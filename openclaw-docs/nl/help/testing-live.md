---
read_when:
    - Live modelmatrix-/CLI-backend-/ACP-/mediaprovidersmoketests uitvoeren
    - Foutopsporing voor de resolutie van inloggegevens bij livetests
    - Een nieuwe providerspecifieke livetest toevoegen
sidebarTitle: Live tests
summary: 'Live-tests (met netwerktoegang): modelmatrix, CLI-backends, ACP, mediaproviders, inloggegevens'
title: 'Testen: livesuites'
x-i18n:
    generated_at: "2026-07-27T05:16:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8279e734e3aa09dd1fa184806c925e0404edfa9acf0f682f73a4955ed90b8b
    source_path: help/testing-live.md
    workflow: 16
---

Voor een snelle start, QA-runners, unit-/integratiesuites en Docker-flows, zie
[Tests](/nl/help/testing). Deze pagina behandelt **live** tests (met netwerktoegang):
modelmatrix, CLI-backends, ACP, mediaproviders en verwerking van inloggegevens.

## Live tests versus je echte Gateway

Live suites en ad-hocrooktests mogen nooit een Gateway verstoren die al
echt verkeer verwerkt (van jou of een andere beheerder):

- Gebruik je eigen Gateway: gebruik de Gateway in hetzelfde proces (laag 2 hieronder) of start een
  ontwikkelinstantie met een geïsoleerde statusmap (`OPENCLAW_STATE_DIR=<scratch>`) en een
  vrije poort. Bind niet aan de standaardpoort van de Gateway (18789) terwijl daarop een echte Gateway
  actief is.
- Voer geen `openclaw gateway stop`/`restart` (of `launchctl`/`systemctl`/tmux-
  equivalenten) uit op een service die je niet in deze sessie hebt gestart — dat is de
  live instantie van de beheerder. Vraag eerst uitdrukkelijke toestemming.
- Realistische gegevens nodig? Kopieer de live status/database naar je ontwikkelstatusmap en test
  met de kopie. Voor rechtstreekse migraties van de status van een live Gateway is eveneens
  uitdrukkelijke toestemming vereist.

## Live: lokale rooktestopdrachten

Exporteer vóór ad-hoc-livecontroles de benodigde providersleutel naar de
procesomgeving.

Veilige mediarooktest:

```bash
pnpm openclaw infer tts convert --local --json \
  --text "OpenClaw live-rooktest." \
  --output /tmp/openclaw-live-smoke.mp3
```

Veilige rooktest voor gereedheid van spraakoproepen:

```bash
pnpm openclaw voicecall setup --json
pnpm openclaw voicecall smoke --to "+15555550123"
```

`voicecall smoke` is een proefuitvoering, tenzij `--yes` ook aanwezig is; gebruik `--yes` alleen
wanneer je daadwerkelijk wilt bellen. Voor Twilio, Telnyx en Plivo vereist een
geslaagde gereedheidscontrole een openbare Webhook-URL; lokale/private
loopback-URL's worden geweigerd omdat die providers ze niet kunnen bereiken.

## Live: controle van Android-Node-mogelijkheden

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Script: `pnpm android:test:integration`
- Doel: roep **elke opdracht aan die momenteel wordt aangeboden** door een verbonden Android-Node en controleer het opdrachtcontract.
- Bereik:
  - Vooraf geconfigureerde/handmatige installatie (de suite installeert, start of koppelt de app niet).
  - Opdrachtgewijze Gateway-validatie van `node.invoke` voor de geselecteerde Android-Node.
- Vereiste voorafgaande configuratie:
  - Android-app is al verbonden met en gekoppeld aan de Gateway.
  - App blijft op de voorgrond.
  - Toestemmingen/toestemming voor opname verleend voor de mogelijkheden waarvan je verwacht dat ze slagen.
- Optionele doeloverschrijvingen:
  - `OPENCLAW_ANDROID_NODE_ID` of `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Volledige details over de Android-configuratie: [Android-app](/nl/platforms/android)

## Live: modelrooktest (profielsleutels)

Live modeltests zijn opgesplitst in twee lagen, zodat fouten geïsoleerd blijven:

- 'Direct model' geeft aan of de provider/het model überhaupt met de opgegeven sleutel kan antwoorden.
- 'Gateway-rooktest' geeft aan of de volledige Gateway- en agentpijplijn voor dat model werkt (sessies, geschiedenis, tools, sandboxbeleid enzovoort).

De beheerde modellijsten hieronder bevinden zich in `src/agents/live-model-filter.ts` en
veranderen in de loop van de tijd; beschouw de arrays daar als de bron van waarheid, niet deze
pagina.

MiniMax M3 gebruikt `minimax/MiniMax-M3` als standaardreferentie voor provider/model.

### Laag 1: directe modelaanvulling (zonder Gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Doel:
  - Gevonden modellen opsommen
  - Gebruik `getApiKeyForModel` om modellen te selecteren waarvoor je inloggegevens hebt
  - Voer per model een kleine aanvulling uit (en waar nodig gerichte regressietests)
- Inschakelen:
  - `pnpm test:live` (of `OPENCLAW_LIVE_TEST=1` als je Vitest rechtstreeks aanroept)
  - Stel `OPENCLAW_LIVE_MODELS=modern`, `small` of `all` (alias voor `modern`) in om deze suite daadwerkelijk uit te voeren; anders wordt deze overgeslagen, zodat `pnpm test:live` op zichzelf gericht blijft op de Gateway-rooktest.
- Modellen selecteren:
  - `OPENCLAW_LIVE_MODELS=modern` voert de beheerde prioriteitenlijst met veelzeggende resultaten uit (zie [Live: modelmatrix](#live-model-matrix-what-we-cover))
  - `OPENCLAW_LIVE_MODELS=small` voert de beheerde prioriteitenlijst voor kleine modellen uit
  - `OPENCLAW_LIVE_MODELS=all` is een alias voor `modern`
  - of `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,..."` (door komma's gescheiden acceptatielijst)
  - Lokale uitvoeringen met kleine Ollama-modellen gebruiken standaard `http://127.0.0.1:11434`; stel `OPENCLAW_LIVE_OLLAMA_BASE_URL` alleen in voor LAN-, aangepaste of Ollama Cloud-eindpunten.
  - Moderne/volledige en kleine controles gebruiken standaard de lengte van hun beheerde lijst als limiet; stel `OPENCLAW_LIVE_MAX_MODELS=0` in voor een volledige controle van het geselecteerde profiel of op een positief getal voor een lagere limiet.
  - Volledige controles gebruiken `OPENCLAW_LIVE_TEST_TIMEOUT_MS` als time-out voor de volledige directe-modeltest. Standaard: 60 minuten.
  - Directe-modelprobes worden standaard met 20-voudige parallelliteit uitgevoerd; stel `OPENCLAW_LIVE_MODEL_CONCURRENCY` in om dit te overschrijven.
- Providers selecteren:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (door komma's gescheiden acceptatielijst)
- Herkomst van sleutels:
  - Standaard: profielopslag en terugvalopties uit de omgeving
  - Stel `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` in om uitsluitend **profielopslag** af te dwingen
- Waarom dit bestaat:
  - Maakt onderscheid tussen 'provider-API is defect/sleutel is ongeldig' en 'Gateway-agentpijplijn is defect'
  - Bevat kleine, geïsoleerde regressies (voorbeeld: opnieuw afspelen van redeneringen en toolaanroepflows van OpenAI Responses/Codex Responses)

### Laag 2: Gateway + rooktest met ontwikkelagent (wat '@openclaw' daadwerkelijk doet)

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Doel:
  - Start een Gateway in hetzelfde proces
  - Maak/wijzig een `agent:dev:*`-sessie (modeloverschrijving per uitvoering)
  - Doorloop modellen-met-sleutels en controleer:
    - 'betekenisvol' antwoord (zonder tools)
    - een echte toolaanroep werkt (leesprobe)
    - optionele aanvullende toolprobes (uitvoer- en leesprobe)
    - OpenAI-regressiepaden (alleen toolaanroep -> vervolgstap) blijven werken
- Probedetails (zodat je fouten snel kunt verklaren):
  - `read`-probe: de test schrijft een noncebestand in de werkruimte en vraagt de agent om dit via `read` te lezen en de nonce terug te geven.
  - `exec+read`-probe: de test vraagt de agent om via `exec` een nonce naar een tijdelijk bestand te schrijven en dit vervolgens via `read` terug te lezen.
  - afbeeldingsprobe: de test voegt een gegenereerde PNG toe (kat + willekeurige code) en verwacht dat het model `cat <CODE>` retourneert.
  - Implementatiereferentie: `src/gateway/gateway-models.profiles.live.test.ts` en `test/helpers/live-image-probe.ts`.
- Inschakelen:
  - `pnpm test:live` (of `OPENCLAW_LIVE_TEST=1` als je Vitest rechtstreeks aanroept)
- Modellen selecteren:
  - Standaard: de beheerde prioriteitenlijst met veelzeggende resultaten (`modern`)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small` voert de beheerde lijst met kleine modellen uit via de volledige Gateway- en agentpijplijn
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` is een alias voor `modern`
  - Of stel `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (of een door komma's gescheiden lijst) in om de selectie te beperken
  - Moderne/volledige en kleine Gateway-controles gebruiken standaard de lengte van hun beheerde lijst als limiet; stel `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` in voor een volledige geselecteerde controle of op een positief getal voor een lagere limiet.
- Providers selecteren (vermijd 'alles via OpenRouter'):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (door komma's gescheiden acceptatielijst)
- Tool- en afbeeldingsprobes zijn in deze live test altijd ingeschakeld:
  - `read`-probe + `exec+read`-probe (toolbelasting)
  - afbeeldingsprobe wordt uitgevoerd wanneer het model ondersteuning voor afbeeldingsinvoer aangeeft
  - Flow (op hoofdlijnen):
    - Test genereert een kleine PNG met 'CAT' + willekeurige code (`test/helpers/live-image-probe.ts`)
    - Verstuurt deze via `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway verwerkt bijlagen tot `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Ingebedde agent stuurt een multimodaal gebruikersbericht door naar het model
    - Controle: antwoord bevat `cat` + de code (OCR-tolerantie: kleine fouten toegestaan)

<Tip>
Voer het volgende uit om te zien wat je op je machine kunt testen (en wat de exacte `provider/model`-id's zijn):

```bash
openclaw models list
openclaw models list --json
```

</Tip>

## Live: rooktest voor CLI-backend (Claude, Gemini of andere lokale CLI's)

- Test: `src/gateway/gateway-cli-backend.live.test.ts`
- Doel: valideer de Gateway- en agentpijplijn met een lokale CLI-backend, zonder je standaardconfiguratie te wijzigen.
- Backendspecifieke standaardwaarden voor rooktests staan bij de `cli-backend.ts`-definitie van de verantwoordelijke Plugin.
- Inschakelen:
  - `pnpm test:live` (of `OPENCLAW_LIVE_TEST=1` als je Vitest rechtstreeks aanroept)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Standaardwaarden:
  - Standaardprovider/-model: `claude-cli/claude-sonnet-4-6`
  - Het gedrag voor opdrachten, argumenten en afbeeldingen komt uit de metadata van de verantwoordelijke CLI-backend-Plugin.
- Overschrijvingen (optioneel):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","json"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` om een echte afbeeldingsbijlage te verzenden (paden worden in de prompt ingevoegd). Standaard uitgeschakeld in Docker-recepten.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` om afbeeldingsbestandspaden als CLI-argumenten door te geven in plaats van ze in de prompt in te voegen.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (of `"list"`) om te bepalen hoe afbeeldingsargumenten worden doorgegeven wanneer `IMAGE_ARG` is ingesteld.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` om een tweede beurt te verzenden en de hervattingsflow te valideren.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=1` om de continuïteitsprobe voor Claude Sonnet -> Opus binnen dezelfde sessie in te schakelen wanneer het geselecteerde model een wisseldoel ondersteunt. Standaard uitgeschakeld, ook in Docker-recepten.
  - `OPENCLAW_LIVE_CLI_BACKEND_MCP_PROBE=1` om de MCP-/tool-loopbackprobe in te schakelen. Standaard uitgeschakeld in Docker-recepten.

Voorbeeld:

```bash
  OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Eenvoudige rooktest voor Gemini MCP-configuratie:

```bash
OPENCLAW_LIVE_TEST=1 \
  pnpm test:live src/agents/cli-runner/bundle-mcp.gemini.live.test.ts
```

Hierbij wordt Gemini niet gevraagd om een antwoord te genereren. De test schrijft dezelfde systeeminstellingen
die OpenClaw aan Gemini geeft en voert vervolgens `gemini --debug mcp list` uit om aan te tonen dat een
opgeslagen `transport: "streamable-http"`-server wordt genormaliseerd naar Gemini's HTTP MCP-
vorm en verbinding kan maken met een lokale streamable-HTTP MCP-server.

Docker-recept:

```bash
pnpm test:docker:live-cli-backend
```

Docker-recepten voor één provider:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:gemini
```

Opmerkingen:

- De Docker-runner bevindt zich op `scripts/test-live-cli-backend-docker.sh`.
- Deze voert de live rooktest voor de CLI-backend binnen de Docker-image van de repository uit als de niet-rootgebruiker `node`.
- Deze haalt de metadata voor de CLI-rooktest op uit de verantwoordelijke plugin en installeert vervolgens het bijbehorende Linux-CLI-pakket (`@anthropic-ai/claude-code` of `@google/gemini-cli`) in een beschrijfbaar voorvoegsel met cache op `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (standaard: `~/.cache/openclaw/docker-cli-tools`).
- `codex-cli` is niet langer een gebundelde CLI-backend; gebruik in plaats daarvan `openai/*` met de Codex-app-serverruntime (zie [Live: rooktest voor de Codex-app-serverharness](#live-codex-app-server-harness-smoke)).
- `pnpm test:docker:live-cli-backend:claude-subscription` vereist overdraagbare OAuth voor een Claude Code-abonnement via `~/.claude/.credentials.json` met `claudeAiOauth.subscriptionType` of via `CLAUDE_CODE_OAUTH_TOKEN` uit `claude setup-token`. Eerst wordt directe `claude -p` in Docker aangetoond, waarna twee Gateway-beurten voor de CLI-backend worden uitgevoerd zonder omgevingsvariabelen met Anthropic-API-sleutels te behouden. Deze abonnementsroute schakelt de Claude MCP-/tool- en afbeeldingsprobes standaard uit, omdat deze de gebruikslimieten van het aangemelde abonnement verbruiken en Anthropic het facturerings- en snelheidslimietgedrag van de Claude Agent SDK / `claude -p` kan wijzigen zonder een OpenClaw-release.
- Claude en Gemini ondersteunen via de bovenstaande vlaggen dezelfde reeks probes (tekstbeurt, afbeeldingsclassificatie, aanroep van de MCP-tool `cron`, continuïteit bij modelwisseling), maar geen van deze probes wordt standaard uitgevoerd. Schakel ze per vlag in wanneer nodig.

## Live: bereikbaarheid van APNs via een HTTP/2-proxy

- Test: `src/infra/push-apns-http2.live.test.ts`
- Doel: via een lokale HTTP CONNECT-proxy een tunnel naar Apples sandbox-APNs-eindpunt maken, het HTTP/2-validatieverzoek van APNs verzenden en controleren of Apples echte `403 InvalidProviderToken`-antwoord via het proxypad terugkomt.
- Inschakelen:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_APNS_REACHABILITY=1 pnpm test:live src/infra/push-apns-http2.live.test.ts`
- Optionele time-out:
  - `OPENCLAW_LIVE_APNS_TIMEOUT_MS=30000`

## Live: ACP-bindingsrooktest (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Doel: de echte ACP-flow voor het binden van gesprekken valideren met een live ACP-agent:
  - verzend `/acp spawn <agent> --bind here`
  - bind ter plekke een synthetisch gesprek via een berichtkanaal
  - verzend een normaal vervolgbericht in datzelfde gesprek
  - controleer of het vervolgbericht in het transcript van de gebonden ACP-sessie terechtkomt
- Inschakelen:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Standaardwaarden:
  - ACP-agents in Docker: `claude,codex,gemini`
  - ACP-agent voor directe `pnpm test:live ...`: `claude`
  - Synthetisch kanaal: gesprekscontext in de stijl van een Slack-DM
  - ACP-backend: `acpx`
- Overschrijvingen:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=droid`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=opencode`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
  - `OPENCLAW_LIVE_ACP_BIND_CODEX_MODEL=gpt-5.6-luna`
  - `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL=opencode/kimi-k2.6`
  - `OPENCLAW_LIVE_ACP_BIND_IMAGE_PROBE=1` (of `on`/`true`/`yes`) om de afbeeldingsprobe geforceerd in te schakelen; elke andere waarde schakelt deze geforceerd uit. Wordt standaard uitgevoerd voor elke agent behalve `opencode`.
  - `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1`
  - `OPENCLAW_LIVE_ACP_BIND_PARENT_MODEL=openai/gpt-5.6-luna`
- Opmerkingen:
  - Deze route gebruikt het Gateway-oppervlak `chat.send` met uitsluitend voor beheerders bestemde synthetische velden voor de oorspronkelijke route, zodat tests context van berichtkanalen kunnen koppelen zonder te doen alsof er extern wordt afgeleverd.
  - Wanneer `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` niet is ingesteld, gebruikt de test het ingebouwde agentregister van de ingesloten plugin `acpx` voor de geselecteerde ACP-harnessagent.
  - Het maken van een gebonden-sessie-Cron via MCP gebeurt standaard naar beste vermogen, omdat externe ACP-harnassen MCP-aanroepen kunnen annuleren nadat het bewijs voor de binding/afbeelding is geslaagd; stel `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1` in om deze Cron-probe na de binding strikt te maken.

Voorbeeld:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker-recept:

```bash
pnpm test:docker:live-acp-bind
```

Docker-recepten voor één agent:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:droid
pnpm test:docker:live-acp-bind:gemini
pnpm test:docker:live-acp-bind:opencode
```

Docker-opmerkingen:

- De Docker-runner bevindt zich op `scripts/test-live-acp-bind-docker.sh`.
- Standaard voert deze de ACP-bindingsrooktest achtereenvolgens uit tegen de gezamenlijke live CLI-agents: `claude`, `codex` en vervolgens `gemini`.
- Gebruik `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=droid`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` of `OPENCLAW_LIVE_ACP_BIND_AGENTS=opencode` om de matrix te beperken.
- Deze plaatst het bijbehorende CLI-authenticatiemateriaal in de container en installeert vervolgens, indien ontbrekend, de gevraagde live CLI (`@anthropic-ai/claude-code`, `@openai/codex`, Factory Droid via `https://app.factory.ai/cli`, `@google/gemini-cli` of `opencode-ai`). De ACP-backend zelf is het ingesloten pakket `acpx/runtime` uit de officiële plugin `acpx`.
- De Droid-Docker-variant plaatst `~/.factory` voor instellingen, geeft `FACTORY_API_KEY` door en vereist die API-sleutel, omdat lokale Factory-OAuth-/sleutelbosauthenticatie niet overdraagbaar is naar de container. Deze gebruikt de ingebouwde registervermelding `droid exec --output-format acp` van ACPX.
- De OpenCode-Docker-variant is een strikte regressieroute voor één agent. Deze schrijft een tijdelijk standaardmodel `OPENCODE_CONFIG_CONTENT` uit `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL` (standaard `opencode/kimi-k2.6`).
- Directe aanroepen van de CLI `acpx` zijn uitsluitend een handmatig/alternatief pad om gedrag buiten de Gateway te vergelijken. De Docker-rooktest voor ACP-binding gebruikt de ingesloten runtimebackend `acpx` van OpenClaw.

## Live: rooktest voor de Codex-app-serverharness

- Doel: de plugin-eigen Codex-harness valideren via de normale Gateway-
  methode `agent`:
  - laad de gebundelde plugin `codex`
  - selecteer een OpenAI-model via `/model <ref> --runtime codex`
  - verzend een eerste Gateway-agentbeurt met het gevraagde denkniveau
  - verzend een tweede beurt naar dezelfde OpenClaw-sessie en controleer of de
    app-serverthread kan worden hervat
  - voer `/codex status` en `/codex models` uit via hetzelfde Gateway-opdrachtpad
  - voer optioneel twee door Guardian beoordeelde shell-probes met verhoogde rechten uit: één onschuldige
    opdracht die moet worden goedgekeurd en één upload met een nepgeheim die moet worden
    geweigerd, zodat de agent om bevestiging vraagt
- Test: `src/gateway/gateway-codex-harness.live.test.ts`
- Inschakelen: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Basismodel van de harness: `openai/gpt-5.6-luna`
- Standaardselectie voor een nieuwe OpenAI-API-sleutel: `openai/gpt-5.6`
- Standaarddenkniveau: `low`
- Modeloverschrijving: `OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/<model>`
- Overschrijving van denkniveau: `OPENCLAW_LIVE_CODEX_HARNESS_THINKING=<level>`
- Controle van inspanningsniveau voor een niet-standaardmodel:
  `OPENCLAW_LIVE_CODEX_HARNESS_EXPECTED_EFFORT=<level>`
- Matrixoverschrijving: `OPENCLAW_LIVE_CODEX_HARNESS_TARGETS=<model>=<thinking>,...`
- Authenticatiemodus: `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=codex-auth` (standaard) gebruikt de
  gekopieerde Codex-aanmelding; `api-key` gebruikt `OPENAI_API_KEY` via de Codex-app-server.
- Optionele afbeeldingsprobe: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- Optionele MCP-/toolprobe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Optionele Guardian-probe: `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1`
- Optionele hervattingsstresstest: `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1` voegt
  vier geschiedenisbeurten toe en sluit en herstart vervolgens de Gateway en Codex-app-server
  drie keer, waarbij dezelfde systeemeigen thread-id en gespreksgeschiedenis
  vereist blijven. Overschrijf de begrensde aantallen met
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_HISTORY_TURNS` (1-20) en
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_RESTARTS` (1-10).
- Optionele uitwaaierstresstest: stel `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1`
  en `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT` (1-12) in. De harness start
  elk onderliggend proces gelijktijdig, wacht op elke definitieve uitvoering en controleert elk
  uniek antwoord van een onderliggend proces en elke systeemeigen thread-identiteit.
- Optionele Compaction-stresstest: `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1`
  genereert begrensde uitvoer van systeemeigen tools, vereist automatische Compaction-gebeurtenissen,
  controleert het opgeslagen aantal Compaction-bewerkingen en het terughalen van verborgen markeringen, herstart
  de Gateway en de fysieke Codex-app-server en herhaalt vervolgens de uitvoer- en
  Compaction-golf. Stem de begrensde werklast af met
  `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS` (1-8) en
  `OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES` (100000-800000).
- Volledige directe-API-context: `OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1` past
  de contextlimiet `922000` en de totale Compaction-limiet `700000` toe, verzendt compacte begrensde
  gebruikersbeurten, voert per golf twee expliciete systeemeigen Compaction-controlepunten uit en
  gaat na elk controlepunt verder met latere beurten. Dit vereist
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` plus een absoluut
  pad `OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG`. De catalogus moet het
  geselecteerde model beschikbaar stellen met `max_context_window: 922000`, zodat Codex de
  overschrijving niet terugbrengt tot het normale catalogusvenster. De gewone stresstest met verlaagde drempelwaarden
  hierboven behoudt de strengere controles voor automatische Compaction en
  het vasthouden van verborgen markeringen.
- Optionele probe voor afmelding van lusdoorgifte:
  `OPENCLAW_LIVE_CODEX_HARNESS_DISABLE_LOOP_RELAY=1`
- De gevraagde denkvoorkeur kan worden toegewezen aan het dichtstbijzijnde inspanningsniveau dat
  Codex voor dat model aanbiedt. Luna wijst bijvoorbeeld `minimal` toe aan `low`.
- Bekende Codex-catalogusmodellen leiden dat exacte systeemeigen inspanningsniveau automatisch af.
  Bij overschrijvingen met onbekende modellen moet het verwachte toegewezen inspanningsniveau worden vermeld.
- De rooktest dwingt provider/model `agentRuntime.id: "codex"` af, zodat een defecte Codex-
  harness niet kan slagen door ongemerkt terug te vallen op OpenClaw.
- Authenticatie: authenticatie van de Codex-app-server via de lokale aanmelding voor het Codex-abonnement, of
  `OPENAI_API_KEY` wanneer `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key`. Docker kan
  `~/.codex/auth.json` en `~/.codex/config.toml` kopiëren voor uitvoeringen met een abonnement.

Lokaal recept:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-luna \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker-recept:

```bash
pnpm test:docker:live-codex-harness
```

Stresstest voor herstarten en geschiedenis:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
pnpm test:docker:live-codex-harness
```

Stresstest voor uitwaaiering, grote uitvoer, Compaction en herstarten:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1 \
  pnpm test:docker:live-codex-harness
```

Compaction-stresstest voor het volledige invoerbudget van systeemeigen Codex `922000`:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG=/absolute/path/to/models-api-1m.json \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-terra \
  OPENCLAW_LIVE_CODEX_HARNESS_THINKING=medium \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES=800000 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Systeemeigen Codex-matrix voor GPT-5.6:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_TARGETS='openai/gpt-5.6-sol=ultra,openai/gpt-5.6-terra=ultra,openai/gpt-5.6-luna=max' \
  pnpm test:docker:live-codex-harness
```

## Live: herhaalde Compaction met OpenAI

- Doel: voer de ingebedde OpenClaw `openai-responses`-agentlus uit via ten
  minste twee echte automatische compactions en controleer vervolgens of een duurzame markering behouden blijft.
- Test: `src/agents/sessions/agent-session.openai-compaction.live.test.ts`
- Inschakelen: `OPENCLAW_LIVE_OPENAI_COMPACTION=1`
- Standaardmodel: `gpt-5.6-luna`
- Modeloverschrijving: `OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=<model>`
- De normale stresstestmodus gebruikt een kleiner clientcontextbudget om met begrensde
  API-kosten hetzelfde echte compaction-pad te bereiken.
- De volledige-contextmodus stelt het clientbudget in op `922000` en de compaction-reserve op
  `222000`, zodat automatische compaction begint bij `700000`. Deze modus vereist ook
  een waargenomen invoeraantal van de provider boven de `272000`-prijsgrens voor lange context.

Begrensd live-recept:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

Recept met volledig `922000`-invoerbudget:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_FULL=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=gpt-5.6-terra \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

<Warning>
De volledige modus overschrijdt bewust OpenAI's prijsgrens voor lange context en
kan meerdere grote API-aanroepen uitvoeren. Gebruik deze modus alleen met expliciete goedkeuring van de kosten.
</Warning>

Standaardwaarde voor een nieuwe OpenAI API-sleutel:

```bash
OPENCLAW_LIVE_GATEWAY_OPENAI_API_DEFAULT=1 \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_THINKING=off \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

Dit bewijs laat `OPENCLAW_LIVE_GATEWAY_MODELS` oningesteld, bepaalt het model via
de nieuwe selectienaad voor afleiding tijdens onboarding, controleert `openai/gpt-5.6` en
voert vervolgens een echte Gateway-beurt uit met dat bepaalde model.

GPT-5.6-matrix voor ingebedde OpenClaw:

```bash
OPENCLAW_LIVE_GATEWAY_THINKING=ultra \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_MODELS='openai/gpt-5.6-sol,openai/gpt-5.6-terra,openai/gpt-5.6-luna' \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

Docker-opmerkingen:

- De Docker-runner bevindt zich in `scripts/test-live-codex-harness-docker.sh`.
- Deze geeft `OPENAI_API_KEY` door, kopieert Codex CLI-authenticatiebestanden wanneer die aanwezig zijn, installeert
  `@openai/codex` in een schrijfbaar aangekoppeld npm-
  voorvoegsel, bereidt de bronstructuur voor en voert vervolgens alleen de live-test van de Codex-harness uit.
- Docker schakelt de probes voor afbeeldingen, MCP/tools en Guardian standaard in. Stel
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` of
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` of
  `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0` in wanneer je een beperktere foutopsporingsrun
  nodig hebt.
- Docker gebruikt dezelfde expliciete Codex-runtimeconfiguratie, zodat verouderde aliassen of OpenClaw-
  fallback een regressie in de Codex-harness niet kunnen verbergen.
- Matrixdoelen worden achtereenvolgens in één container uitgevoerd. Het Docker-script schaalt zijn
  standaardtime-out van 35 minuten met het aantal doelen; elke time-out van een buitenliggende shell of CI moet
  dezelfde totale tijd toestaan. Canonieke CI houdt elk GPT-5.6-doel in een afzonderlijke shard.

### Aanbevolen live-recepten

Smalle, expliciete toelatingslijsten zijn het snelst en het minst instabiel:

- Eén model, rechtstreeks (zonder Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna" pnpm test:live src/agents/models.profiles.live.test.ts`

- Rechtstreeks profiel voor een klein model:
  - `OPENCLAW_LIVE_MODELS=small pnpm test:live src/agents/models.profiles.live.test.ts`

- Gateway-profiel voor een klein model:
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Ollama Cloud API-rooktest:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 pnpm test:live -- extensions/ollama/ollama.live.test.ts`

- Eén model, Gateway-rooktest:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Toolaanroepen bij meerdere providers:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.5-flash,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Rechtstreekse rooktest voor Z.AI Coding Plan GLM-5.2:
  - `ZAI_CODING_LIVE_TEST=1 pnpm test:live src/agents/zai.live.test.ts`

- Google-focus (Gemini API-sleutel + Antigravity):
  - Gemini (API-sleutel): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3.5-flash" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google-rooktest voor adaptief denken (`qa manual` vanuit de privé-QA-CLI — vereist `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` en een broncheckout; zie [QA-overzicht](/nl/concepts/qa-e2e-automation)):
  - Dynamische standaardwaarde van Gemini 3: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-3.1-pro-preview --alt-model google/gemini-3.1-pro-preview --message '/think adaptive Reply exactly: GEMINI_ADAPTIVE_OK' --timeout-ms 180000`
  - Dynamisch budget van Gemini 2.5: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-2.5-flash --alt-model google/gemini-2.5-flash --message '/think adaptive Reply exactly: GEMINI25_ADAPTIVE_OK' --timeout-ms 180000`

Opmerkingen:

- `google/...` gebruikt de Gemini API (API-sleutel).
- `google-antigravity/...` gebruikt de Antigravity OAuth-bridge (agentendpoint in Cloud Code Assist-stijl).
- `google-gemini-cli/...` gebruikt de lokale Gemini CLI op je computer (afzonderlijke authenticatie en eigenaardigheden van de tooling).
- Gemini API versus Gemini CLI:
  - API: OpenClaw roept Google's gehoste Gemini API aan via HTTP (API-sleutel/profielauthenticatie); dit is wat de meeste gebruikers met "Gemini" bedoelen.
  - CLI: OpenClaw voert een lokaal `gemini`-binair bestand uit via de shell; dit heeft eigen authenticatie en kan zich anders gedragen (streaming/toolondersteuning/versieverschillen).

## Live: modelmatrix (wat we dekken)

Live is optioneel, dus er is geen vaste "CI-modellenlijst". `OPENCLAW_LIVE_MODELS=modern` / `OPENCLAW_LIVE_GATEWAY_MODELS=modern` (en hun alias `all`) voeren de samengestelde prioriteitenlijst uit `HIGH_SIGNAL_LIVE_MODEL_PRIORITY` in `src/agents/live-model-filter.ts` uit, in deze prioriteitsvolgorde:

| Provider/model                                | Opmerkingen |
| --------------------------------------------- | ----------- |
| `anthropic/claude-opus-5`                     |             |
| `anthropic/claude-opus-4-8`                   |             |
| `anthropic/claude-sonnet-5`                   |             |
| `anthropic/claude-sonnet-4-6`                 |             |
| `anthropic/claude-opus-4-7`                   |             |
| `google/gemini-3.1-pro-preview`               | Gemini API  |
| `google/gemini-3.5-flash`                     | Gemini API  |
| `cohere/command-a-plus-05-2026`               |             |
| `moonshot/kimi-k3`                            |             |
| `anthropic/claude-opus-4-6`                   |             |
| `deepseek/deepseek-v4-flash`                  |             |
| `deepseek/deepseek-v4-pro`                    |             |
| `minimax/MiniMax-M3`                          |             |
| `openai/gpt-5.5`                              |             |
| `openrouter/openai/gpt-5.2-chat`              |             |
| `openrouter/minimax/minimax-m2.7`             |             |
| `opencode-go/glm-5`                           |             |
| `openrouter/ai21/jamba-large-1.7`             |             |
| `xai/grok-4.5`                                |             |
| `xai/grok-4.20-0309-reasoning`                |             |
| `zai/glm-5.1`                                 |             |
| `fireworks/accounts/fireworks/models/glm-5p1` |             |
| `minimax-portal/minimax-m3`                   |             |

De samengestelde lijst met **kleine modellen** (`OPENCLAW_LIVE_MODELS=small` / `OPENCLAW_LIVE_GATEWAY_MODELS=small`), uit `SMALL_LIVE_MODEL_PRIORITY`:

| Provider/model               |
| ---------------------------- |
| `lmstudio/qwen/qwen3.5-9b`   |
| `vllm/qwen/qwen3-8b`         |
| `sglang/qwen/qwen3-8b`       |
| `ollama/gemma3:4b`           |
| `openrouter/qwen/qwen3.5-9b` |
| `openrouter/z-ai/glm-5.1`    |
| `openrouter/z-ai/glm-5`      |
| `zai/glm-5.1`                |

Opmerkingen over de moderne lijst:

- De providers `codex` en `codex-cli` zijn uitgesloten van de standaard moderne sweep (ze dekken het gedrag van de CLI-backend/ACP, dat hierboven afzonderlijk wordt getest). `openai/gpt-5.5` zelf routeert standaard via de harness van de Codex-appserver; zie [Live: rooktest voor de harness van de Codex-appserver](#live-codex-app-server-harness-smoke).
- `fireworks`, `google`, `openrouter` en `xai` voeren in de moderne sweep alleen hun expliciet samengestelde model-id's uit (geen automatische uitbreiding naar "elk model van deze provider").
- Neem ten minste één model met afbeeldingsondersteuning (vision-varianten uit de Claude/Gemini/OpenAI-familie enzovoort) op in `OPENCLAW_LIVE_GATEWAY_MODELS` om de afbeeldingsprobe uit te voeren.

Voer een Gateway-rooktest met tools en afbeeldingen uit voor een handmatig gekozen set van verschillende providers:

```bash
OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3.5-flash,google-antigravity/claude-opus-4-6-thinking,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts
```

Optionele aanvullende dekking buiten de samengestelde lijsten (prettig om te hebben; kies een model met ondersteuning voor "tools" dat je hebt ingeschakeld):

- Mistral: `mistral/...`
- Cerebras: `cerebras/...` (als je toegang hebt)
- LM Studio: `lmstudio/...` (lokaal; toolaanroepen zijn afhankelijk van de API-modus)

### Aggregators / alternatieve gateways

Als je sleutels hebt ingeschakeld, kun je ook testen via:

- OpenRouter: `openrouter/...` (honderden modellen; gebruik `openclaw models scan` om kandidaten met ondersteuning voor tools en afbeeldingen te vinden)
- OpenCode: `opencode/...` voor Zen en `opencode-go/...` voor Go (authenticatie via `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Meer providers die je in de live-matrix kunt opnemen (als je referenties/configuratie hebt):

- Ingebouwd: `anthropic`, `cerebras`, `github-copilot`, `google`, `google-antigravity`, `google-gemini-cli`, `google-vertex`, `groq`, `mistral`, `openai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `zai`
- Via `models.providers` (aangepaste endpoints): `minimax` (cloud/API), plus elke OpenAI-/Anthropic-compatibele proxy (LM Studio, vLLM, LiteLLM enzovoort)

<Tip>
Codeer niet "alle modellen" hard in documentatie. De gezaghebbende lijst is wat `discoverModels(...)` op je computer retourneert, aangevuld met de beschikbare sleutels.
</Tip>

## Referenties (nooit committen)

Live-tests vinden referenties op dezelfde manier als de CLI. Praktische gevolgen:

- Als de CLI werkt, zouden live-tests dezelfde sleutels moeten vinden.
- Als een live-test "no creds" meldt, spoor je dit op dezelfde manier op als `openclaw models list` / modelselectie.

- Authenticatieprofielen per agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (dit wordt in de live-tests bedoeld met "profile keys")
- Configuratie: `~/.openclaw/openclaw.json` (of `OPENCLAW_CONFIG_PATH`)
- Verouderde OAuth-map: `~/.openclaw/credentials/` (wordt indien aanwezig naar de voorbereide live-homemap gekopieerd, maar is niet de primaire opslag voor profielsleutels)
- Lokale live-runs kopiëren de actieve configuratie (waarbij overschrijvingen van `agents.*.workspace` / `agentDir` zijn verwijderd) en de `auth-profiles.json` van elke agent — niet de rest van de map van die agent, zodat gegevens uit `workspace/` en `sandboxes/` nooit in de voorbereide homedirectory terechtkomen — plus de verouderde map `credentials/` en ondersteunde authenticatiebestanden/-mappen van externe CLI's (`.claude.json`, `.claude/.credentials.json`, `.claude/settings*.json`, `.claude/backups`, `.codex/auth.json`, `.codex/config.toml`, `.gemini`, `.minimax`) naar een tijdelijke test-homedirectory.

Als je omgevingssleutels wilt gebruiken, exporteer je ze vóór lokale tests of gebruik je de
onderstaande Docker-runners met een expliciete `OPENCLAW_PROFILE_FILE`.

## Deepgram live (audiotranscriptie)

- Test: `extensions/deepgram/audio.live.test.ts`
- Inschakelen: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live extensions/deepgram/audio.live.test.ts`

## BytePlus-coderingsplan live

- Test: `extensions/byteplus/live.test.ts`
- Inschakelen: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live extensions/byteplus/live.test.ts`
- Optionele modeloverschrijving: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI-workflowmedia live

- Test: `extensions/comfy/comfy.live.test.ts`
- Inschakelen: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Bereik:
  - Voert de gebundelde comfy-paden voor afbeeldingen, video en `music_generate` uit
  - Slaat elke mogelijkheid over tenzij `plugins.entries.comfy.config.<capability>` is geconfigureerd
  - Nuttig na wijzigingen aan het indienen, pollen of downloaden van comfy-workflows of aan Plugin-registratie

## Live afbeeldingsgeneratie

- Test: `test/image-generation.runtime.live.test.ts`
- Opdracht: `pnpm test:live test/image-generation.runtime.live.test.ts`
- Testharnas: `pnpm test:live:media image`
- Bereik:
  - Somt elke geregistreerde provider-Plugin voor het genereren van afbeeldingen op
  - Gebruikt reeds geëxporteerde omgevingsvariabelen van providers vóór het testen
  - Gebruikt standaard live/API-sleutels uit de omgeving vóór opgeslagen authenticatieprofielen, zodat verouderde testsleutels in `auth-profiles.json` echte shellreferenties niet maskeren
  - Slaat providers zonder bruikbare authenticatie/profiel/model over
  - Voert elke geconfigureerde provider uit via de gedeelde runtime voor het genereren van afbeeldingen:
    - `<provider>:generate`
    - `<provider>:edit` wanneer de provider ondersteuning voor bewerken declareert
- Huidige meegeleverde providers die worden gedekt:
  - `deepinfra`
  - `fal`
  - `google`
  - `minimax`
  - `openai`
  - `openrouter`
  - `vydra`
  - `xai`
- Optionele beperking:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google,openrouter,xai"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="deepinfra"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-2,google/gemini-3.1-flash-image,openrouter/google/gemini-3.1-flash-image-preview,xai/grok-imagine-image"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit,openrouter:generate,xai:default-generate,xai:default-edit"`
- Optioneel authenticatiegedrag:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` om authenticatie via de profielopslag af te dwingen en uitsluitend via de omgeving ingestelde overschrijvingen te negeren

Voeg voor het meegeleverde CLI-pad een `infer`-smoketest toe nadat de live
test voor de provider/runtime slaagt:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_INFER_CLI_TEST=1 pnpm test:live -- test/image-generation.infer-cli.live.test.ts
openclaw infer image providers --json
openclaw infer image generate \
  --model google/gemini-3.1-flash-image \
  --prompt "Minimale vlakke testafbeelding: één blauw vierkant op een witte achtergrond, zonder tekst." \
  --output ./openclaw-infer-image-smoke.png \
  --json
```

Dit dekt het verwerken van CLI-argumenten, de resolutie van configuratie/standaardagent, activering van
meegeleverde plugins, de gedeelde runtime voor het genereren van afbeeldingen en het live
providerverzoek. De afhankelijkheden van de Plugin moeten aanwezig zijn voordat de runtime wordt geladen.

## Live muziekgeneratie

- Test: `extensions/music-generation-providers.live.test.ts`
- Inschakelen: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Testharnas: `pnpm test:live:media music`
- Bereik:
  - Test het gedeelde pad van de meegeleverde provider voor muziekgeneratie
  - Dekt momenteel `fal`, `google`, `minimax` en `openrouter`
  - Gebruikt reeds geëxporteerde omgevingsvariabelen van providers vóór het testen
  - Gebruikt standaard live/API-sleutels uit de omgeving vóór opgeslagen authenticatieprofielen, zodat verouderde testsleutels in `auth-profiles.json` echte shellreferenties niet maskeren
  - Slaat providers zonder bruikbare authenticatie/profiel/model over
  - Voert beide gedeclareerde runtimemodi uit wanneer deze beschikbaar zijn:
    - `generate` met alleen een prompt als invoer
    - `edit` wanneer de provider `capabilities.edit.enabled` declareert
  - `comfy` heeft een eigen afzonderlijk livebestand en maakt geen deel uit van deze gedeelde testronde
- Optionele beperking:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.6"`
- Optioneel authenticatiegedrag:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` om authenticatie via de profielopslag af te dwingen en uitsluitend via de omgeving ingestelde overschrijvingen te negeren

## Live videogeneratie

- Test: `extensions/video-generation-providers.live.test.ts`
- Inschakelen: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Testharnas: `pnpm test:live:media video`
- Bereik:
  - Test het gedeelde pad van de meegeleverde provider voor videogeneratie voor `alibaba`, `byteplus`, `deepinfra`, `fal`, `google`, `minimax`, `openai`, `openrouter`, `pixverse`, `qwen`, `runway`, `together`, `vydra`, `xai`
  - Gebruikt standaard het releaseveilige smoketestpad: één tekst-naar-videoverzoek per provider, een kreeftenprompt van één seconde en een limiet per providerbewerking uit `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (standaard `180000`)
  - Slaat FAL standaard over omdat wachtrijvertraging aan de kant van de provider de releasetijd kan domineren; geef `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` door (of wis de lijst met over te slaan providers) om deze expliciet uit te voeren
  - Gebruikt reeds geëxporteerde omgevingsvariabelen van providers vóór het testen
  - Gebruikt standaard live/API-sleutels uit de omgeving vóór opgeslagen authenticatieprofielen, zodat verouderde testsleutels in `auth-profiles.json` echte shellreferenties niet maskeren
  - Slaat providers zonder bruikbare authenticatie/profiel/model over
  - Voert standaard alleen `generate` uit
  - Stel `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` in om ook gedeclareerde transformatiemodi uit te voeren wanneer deze beschikbaar zijn:
    - `imageToVideo` wanneer de provider `capabilities.imageToVideo.enabled` declareert en de geselecteerde provider/het geselecteerde model in de gedeelde testronde lokale afbeeldingsinvoer uit een buffer accepteert
    - `videoToVideo` wanneer de provider `capabilities.videoToVideo.enabled` declareert en de geselecteerde provider/het geselecteerde model in de gedeelde testronde lokale video-invoer uit een buffer accepteert
  - Huidige gedeclareerde maar overgeslagen `imageToVideo`-provider in de gedeelde testronde:
    - `vydra` (lokale afbeeldingsinvoer uit een buffer wordt in dit traject niet ondersteund)
  - Providerspecifieke dekking voor Vydra:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - Dat bestand voert `veo3` tekst-naar-video uit plus een `kling`-traject voor afbeelding-naar-video dat standaard een externe afbeeldings-URL als fixture gebruikt (`OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL` om dit te overschrijven).
  - Providerspecifieke dekking voor xAI:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"`
    - De klassieke variant genereert eerst een vierkante lokale PNG als eerste frame, laat geometrie weg, vraagt een afbeelding-naar-videoclip van één seconde aan, peilt tot voltooiing en verifieert de gedownloade buffer.
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"`
    - De 1.5-variant genereert een lokale PNG als eerste frame, vraagt een 1080P-afbeelding-naar-videoclip van één seconde aan, peilt tot voltooiing en verifieert de gedownloade buffer.
  - Huidige live dekking voor `videoToVideo`:
    - `runway` alleen wanneer het geselecteerde model wordt omgezet naar `gen4_aleph`
  - Huidige gedeclareerde maar overgeslagen `videoToVideo`-providers in de gedeelde testronde:
    - `alibaba`, `google`, `openai`, `qwen`, `xai` omdat deze paden momenteel externe `http(s)`-referentie-URL's vereisen in plaats van lokale invoer uit een buffer
- Optionele beperking:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="deepinfra,google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""` om elke provider in de standaardtestronde op te nemen, inclusief FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000` om voor een intensieve smoketest de limiet voor elke providerbewerking te verlagen
- Optioneel authenticatiegedrag:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` om authenticatie via de profielopslag af te dwingen en uitsluitend via de omgeving ingestelde overschrijvingen te negeren

## Live mediaharnas

- Opdracht: `pnpm test:live:media`
- Ingangspunt: `test/e2e/qa-lab/media/hosted-media-provider-live.ts`, dat `pnpm test:live -- <suite-test-file>` per geselecteerde suite uitvoert, zodat Heartbeat- en stillemodusgedrag consistent blijven met andere `pnpm test:live`-uitvoeringen.
- Doel:
  - Voert de gedeelde live suites voor afbeeldingen, muziek en video uit via één repo-eigen ingangspunt
  - Laadt ontbrekende omgevingsvariabelen van providers automatisch uit `~/.profile`
  - Beperkt elke suite standaard automatisch tot providers die momenteel bruikbare authenticatie hebben
- Vlaggen:
  - `--providers <csv>` algemeen providerfilter; `--image-providers` / `--music-providers` / `--video-providers` beperken een filter tot één suite
  - `--all-providers` slaat het automatische filter op basis van authenticatie over
  - `--allow-empty` sluit af met `0` wanneer na het filteren geen uitvoerbare providers overblijven
  - `--quiet` / `--no-quiet` worden doorgegeven aan `test:live`
- Voorbeelden:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Gerelateerd

- [Testen](/nl/help/testing) - unit-, integratie-, QA- en Docker-suites
