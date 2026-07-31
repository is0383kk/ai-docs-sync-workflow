---
read_when:
    - Je wilt inferentie instellen en vervolgens de configuratie voltooien met OpenClaw
summary: CLI-referentie voor `openclaw onboard` (interactieve onboarding)
title: Aan de slag
x-i18n:
    generated_at: "2026-07-27T05:05:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec5cfc564aa14041d1aa67a978a4661e6105b7119a942940f71197c695e788b
    source_path: cli/onboard.md
    workflow: 16
---

# `openclaw onboard`

Begeleide configuratie waarbij inferentie eerst wordt ingesteld: bestaande AI-toegang wordt gedetecteerd,
een live voltooiing is vereist, alleen de werkende route wordt opgeslagen en vervolgens wordt
OpenClaw gestart om de rest te configureren. `openclaw setup` opent deze flow op nieuwe
systemen of wanneer een onboardingoptie aanwezig is; geconfigureerde systemen gebruiken
alleen `openclaw setup` voor chat met de systeemagent. `openclaw setup --baseline`
schrijft alleen de basisconfiguratie/-werkruimte.

<CardGroup cols={2}>
  <Card title="CLI-onboardinghub" href="/nl/start/wizard" icon="rocket">
    Stapsgewijze uitleg van de interactieve CLI-flow.
  </Card>
  <Card title="Onboardingoverzicht" href="/nl/start/onboarding-overview" icon="map">
    Hoe de onboarding van OpenClaw samenhangt.
  </Card>
  <Card title="CLI-configuratiereferentie" href="/nl/start/wizard-cli-reference" icon="book">
    Uitvoer, interne werking en gedrag per stap.
  </Card>
  <Card title="CLI-automatisering" href="/nl/start/wizard-cli-automation" icon="terminal">
    Niet-interactieve vlaggen en gescripte configuraties.
  </Card>
  <Card title="Onboarding van de macOS-app" href="/nl/start/onboarding" icon="apple">
    Onboardingflow voor de macOS-menubalkapp.
  </Card>
</CardGroup>

## Voorbeelden

```bash
openclaw onboard
openclaw onboard --tui
openclaw onboard --classic
openclaw onboard --modern
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --flow import
openclaw onboard --import-from hermes --import-source ~/.hermes
openclaw onboard --skip-bootstrap
openclaw onboard recommendations --json
openclaw onboard recommendations acknowledge
openclaw onboard recommendations acknowledge --retry "<failed-id>"
openclaw onboard recommendations refresh
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

`openclaw onboard recommendations` leest overeenkomsten voor app-aanbevelingen die
tijdens de onboarding zijn opgeslagen. Voeg `--json` toe voor de machineleesbare lijst die door
de bootstrap bij de eerste uitvoering wordt gebruikt. De opdracht scant geïnstalleerde apps niet opnieuw en roept geen
model aan. De uitvoer bevat alleen gevalideerde installatie-ID's, de bron en het niveau; opzettelijk
worden niet-vertrouwde marktplaatsteksten, modelredenen en lokale app-labels
weggelaten. Nadat het aanbod met aanbevelingen is beantwoord, retourneert de opdracht
een lege lijst en slaan toekomstige onboardingsessies de stap volledig over.
`openclaw onboard recommendations refresh` wist het opgeslagen aanbod, zodat tijdens de volgende
onboardingsessie geïnstalleerde apps opnieuw worden gescand en een nieuw aanbod wordt gemaakt.

In nieuwe werkruimten wordt de keuze voor aanbevelingen uitgesteld tot het bootstrapgesprek.
Nadat in dat gesprek de keuzes van de gebruiker zijn verwerkt,
markeert `openclaw onboard recommendations acknowledge` het opgeslagen aanbod als beantwoord.
De bevestiging is idempotent. Als een gekozen installatie mislukt, geef je elk mislukt
ondoorzichtig ID door met `--retry <id...>`; geslaagde en afgewezen overeenkomsten worden verwerkt,
terwijl mislukte overeenkomsten in behandeling blijven voor een latere onboardingsessie. Onbekende ID's
mislukken zonder het opgeslagen aanbod te wijzigen. Na een onderbroken installatie van een ClawHub-skill
geldt een bestaand doel alleen als geslaagd wanneer
`openclaw skills verify "@owner/slug"` slaagt voor hetzelfde
door de uitgever gekwalificeerde aanbevelings-ID en de JSON-uitvoer
`openclaw.resolution.source: "installed"` meldt. Alleen verificatie via het register is geen
bewijs van een lokale installatie. Laat dat ID anders in behandeling met `--retry` en
overschrijf de bestaande skill niet.

- `--classic`: opent de volledige stapsgewijze wizard. Dit kan niet worden gecombineerd met
  `--non-interactive`; laat `--classic` weg voor geautomatiseerde configuratie.
- `--flow quickstart`: opent de klassieke wizard met minimale prompts, gebruikt
  standaard tokenauthenticatie en genereert een token wanneer geen opgeslagen of expliciete
  credential van toepassing is. Expliciete lokale Gateway-vlaggen zoals
  `--gateway-port`, `--gateway-bind`, `--gateway-auth` en `--tailscale`
  overschrijven de bijbehorende opgeslagen of standaard quickstartwaarden; weggelaten
  opties behouden hun huidige waarden.
- `--flow manual` (alias `advanced`): opent de klassieke wizard met volledige prompts
  voor poort, binding en authenticatie.
- `--flow import`: voert een gedetecteerde migratieprovider uit (bijvoorbeeld Hermes via `--import-from hermes`) op een nieuwe configuratie. Na bevestiging zet de onboarding configuratie, credentials, werkruimtebestanden, geheugen en skills klaar onder afgeschermde tijdelijke doelen; geïmporteerde inferentie moet een live voltooiing doorstaan voordat de werkruimte- en agentstatus worden geactiveerd en de configuratie wordt vastgelegd. Een fout of annulering vóór activering laat het actieve doel onaangeroerd. Externe activeringsstappen die niet kunnen worden teruggedraaid, zoals de installatie van een Codex-plugin, worden daarna uitgevoerd en kunnen opnieuw worden geprobeerd vanuit het migratierapport. Stel eerst configuratie, credentials, sessies en werkruimtestatus opnieuw in als deze al bestaan. Gebruik [`openclaw migrate`](/nl/cli/migrate) voor droogloopplannen, overschrijfmodus, geverifieerde back-ups, rapporten en exacte toewijzingen.
- `--remote-url` en `--remote-token`: vullen de klassieke stap voor een externe Gateway vooraf in en overschrijven voor deze uitvoering opgeslagen externe waarden. Als je de URL wijzigt, worden opgeslagen credentials niet hergebruikt, tenzij je ook een token doorgeeft. Het token blijft gemaskeerd in prompts en volgt de bestaande keuze van de wizard voor opslag als platte tekst of SecretRef.
- `--tailscale-reset-on-exit` en `--no-tailscale-reset-on-exit`: bepalen expliciet of de configuratie van Tailscale Serve of Funnel opnieuw wordt ingesteld wanneer de Gateway afsluit. Als je beide weglaat, blijft de huidige instelling behouden tijdens niet-interactieve heruitvoeringen.
- `--modern` is een compatibiliteitsalias voor de conversationele configuratie-
  assistent van OpenClaw. Deze gebruikt dezelfde live-inferentiedrempel als `openclaw setup` en
  accepteert alleen `--workspace`, `--accept-risk`,
  `--non-interactive` en `--json`. Andere configuratievlaggen worden geweigerd in plaats van
  stilzwijgend te worden genegeerd.

## Begeleide flow

Alleen `openclaw onboard` start de begeleide flow. Eerst wordt de beveiligingsmelding getoond,
daarna volgt vooraf één vraag: **volledige toegang** (aanbevolen — de configuratie zoekt
automatisch naar AI-apps, sleutels en lokale runtimes) of **eerst vragen** (de configuratie vraagt
eenmaal toestemming voordat er wordt rondgekeken, of laat je handmatig configureren). De
keuze wordt opgeslagen als `wizard.accessMode`. Wanneer detectie is toegestaan, detecteert de onboarding
AI-toegang die al beschikbaar is via geconfigureerde modellen, omgevingsvariabelen
met API-sleutels en ondersteunde lokale CLI's, en test vervolgens de aanbevolen
kandidaat met een echte voltooiing. Als een kandidaat mislukt, probeert de onboarding ongemerkt
de volgende bruikbare kandidaat en vat alles wat niet reageerde op één
regel samen; de werkende route wordt aangekondigd met een optie waarvoor één toetsaanslag nodig is om
in plaats daarvan alle andere mogelijkheden te bekijken.

Als de automatische detectie geen mogelijkheden meer heeft, toont de providerkiezer eerst OpenAI,
Anthropic, xAI (Grok), Google en OpenRouter. Kies **Meer…** voor elke
andere ondersteunde provider, gegroepeerd per provider; regio's, abonnementen en authenticatiemethoden
verschijnen vervolgens in een tweede menu. Ondersteunde aanmelding via browser of apparaat en gemaskeerde
methoden met API-sleutels of tokens gebruiken hetzelfde pad voor live voltooiing. OpenClaw slaat
pas na een geslaagde test de geverifieerde modelroute en bijbehorende credential op; een
mislukte kandidaat vervangt het geconfigureerde model niet en de gebruikte
credential wordt niet opgeslagen. Kies **Voorlopig overslaan** om af te sluiten zonder OpenClaw te starten en
voer `openclaw onboard` opnieuw uit wanneer je klaar bent. De configuratie van de werkruimte en Gateway blijft
ongewijzigd totdat OpenClaw start.

In de begeleide modus levert `--workspace <dir>` de voorgestelde werkruimte van OpenClaw
en de geïsoleerde inferentiecontext. Deze wordt pas opgeslagen wanneer je het
configuratievoorstel van OpenClaw goedkeurt. Bij klassieke en niet-interactieve onboarding wordt de
werkruimte via de normale configuratieflow opgeslagen. Bij een heruitvoering met een bestaand agentenbestand
behoudt de onboarding de geconfigureerde werkruimte van de vloot: de klassieke
wizard toont beide paden en vereist expliciete bevestiging voordat deze wordt verplaatst,
terwijl de niet-interactieve configuratie een waarschuwing geeft en de huidige waarde behoudt.

Nadat inferentie is geslaagd, controleert de onboarding op herinneringen van ondersteunde lokale AI-
tools: het automatische geheugen van Claude Code, samengevoegde herinneringen van Codex en Hermes-geheugen-
bestanden. Als er iets wordt gevonden, biedt één pagina aan om dit naar de agentwerkruimte te kopiëren
onder `memory/imports/` voor geïndexeerd terugzoeken. Er wordt niets geïmporteerd zonder
bevestiging, eerder geïmporteerde bestanden worden overgeslagen en je kunt altijd
later importeren via de [pagina voor geheugenimport](/nl/web/control-ui) in de Control UI, die
dezelfde scope met uitsluitend geheugen biedt. (Een volledige uitvoering van [`openclaw migrate`](/nl/cli/migrate) is
breder: hiermee kunnen ook configuratie, skills en credentials worden geïmporteerd.) De klassieke
wizard toont dezelfde pagina nadat de werkruimte is voorbereid.

Nadat inferentie is geslaagd (en na het aanbod voor geheugenimport), past de begeleide onboarding
automatisch de standaardconfiguratie toe — werkruimte, Gateway en sessies,
hetzelfde plan dat de conversationele chat van `openclaw setup` zou toepassen bij "ja" —
en biedt vervolgens aanbevelingen voor plugins en skills op basis van geïnstalleerde apps; appnamen
worden vergeleken via je geconfigureerde model en een ClawHub-zoekopdracht, en de stap kan
worden uitgeschakeld met [`wizard.appRecommendations`](/nl/gateway/configuration-reference#wizard).
In een desktopsessie op macOS, Linux of Windows wordt vervolgens het geauthenticeerde
Control UI-dashboard geopend en maximaal 60 seconden gewacht totdat de browserclient
verbinding maakt. Op Linux zonder grafische interface of via SSH wordt een opvallende, kopieerbare
dashboard-URL afgedrukt, inclusief een SSH-opdracht voor poortdoorsturing voor een Gateway
op loopback, en maximaal vijf minuten gewacht. Bij een geslaagde verbinding ga je verder in de browser;
bij een onbereikbare Gateway of time-out wordt teruggevallen op dezelfde terminaluitweg als
voorheen. Geef `--tui` door om de overdracht naar de browser over te slaan en die terminaluitweg af te dwingen.
Als het toepassen van de configuratie mislukt, valt de onboarding terug op de conversationele OpenClaw-
chat om de configuratie interactief te voltooien. Kanalen, agenten,
plugins en andere optionele functies blijven onderdeel van de OpenClaw-chat: voer
`openclaw` uit en gebruik `open channel wizard for <channel>` om het verzamelen van kanaal-
credentials over te dragen aan een gemaskeerde terminalwizard. Als je de model-
provider of bijbehorende authenticatie wilt wijzigen, sluit je OpenClaw af en voer je `openclaw onboard` uit;
OpenClaw opent de begeleide of klassieke providerflows niet.

Bij een geconfigureerde installatie verifieert een nieuwe uitvoering van `openclaw onboard` eerst het huidige
standaardmodel, zodat dezelfde flow als verificatie- en herstelprocedure fungeert —
de configuratie wordt niet opnieuw toegepast en de Gateway-service wordt niet opnieuw geïnstalleerd of gestart.
Als die controle mislukt, wordt het geconfigureerde model nooit automatisch vervangen —
de onboarding stopt en vraagt hoe je wilt doorgaan. De controle wordt buiten je
werkruimte uitgevoerd, waardoor een model dat door een werkruimteplugin wordt geleverd hier kan mislukken terwijl het
in de agent wel werkt.
Gebruik `openclaw onboard --classic` voor providerspecifieke authenticatie, kanalen, skills,
configuratie van een externe Gateway, imports of volledige Gateway-bediening. Voer voor conversationele
configuratie en herstel zonder inferentie `openclaw setup` uit; `openclaw onboard
--modern` is een compatibiliteitsalias via dezelfde inferentiedrempel. De klassieke
wizard kan het standaardmodel optioneel verifiëren met een live voltooiing, maar
OpenClaw start pas nadat de eigen live-inferentiecontrole is geslaagd.

In een interactieve terminal wordt alleen `openclaw` (zonder subopdracht) gerouteerd op basis van de
configuratiestatus:

- Als het actieve configuratiebestand ontbreekt of geen handmatig opgegeven instellingen bevat (leeg of
  alleen metadata), wordt de begeleide onboarding gestart.
- Als het configuratiebestand bestaat maar niet door de validatie komt, wordt het klassieke
  onboardingpad gestart met begeleiding van `openclaw doctor`. OpenClaw vereist werkende
  inferentie en wordt niet gebruikt om deze status vóór inferentie te herstellen.
- Als het configuratiebestand geldig is, wordt de normale agent-TUI geopend. Een bereikbare
  geconfigureerde Gateway met een agent en model gaat rechtstreeks naar die UI zonder
  onboarding of OpenClaw. Bij een geconfigureerde installatie bereik je OpenClaw met
  `/openclaw` in de TUI of `openclaw setup`.

`ws://` als platte tekst wordt geaccepteerd voor loopback, privé-IP-literals, `.local` en Gateway-URL's met Tailnet `*.ts.net`. Stel voor andere vertrouwde namen in privé-DNS `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` in binnen de procesomgeving van de onboarding.

## Opnieuw instellen

```bash
openclaw onboard --reset
openclaw onboard --reset --reset-scope full
```

`--reset` wist de status voordat de installatie wordt uitgevoerd. `--reset-scope` bepaalt hoeveel er wordt gewist: `config` (alleen configuratie), `config+creds+sessions` (standaard wanneer `--reset` zonder bereik wordt doorgegeven) of `full` (stelt ook de werkruimte opnieuw in). De werkruimte wordt alleen opnieuw ingesteld met `--reset-scope full`.

## Landinstelling

Interactieve onboarding gebruikt de landinstelling van de CLI-wizard voor vaste installatieteksten. De eerste niet-lege waarde in deze volgorde wordt gebruikt:

1. `OPENCLAW_LOCALE`
2. `LC_ALL`
3. `LC_MESSAGES`
4. `LANG`
5. Engels als terugvaloptie

Ondersteunde landinstellingen voor de wizard zijn `en`, `zh-CN` en `zh-TW`. Waarden voor landinstellingen mogen een onderstrepingsteken of POSIX-achtervoegsel gebruiken, zoals `zh_CN.UTF-8`. Productnamen, opdrachtnamen, configuratiesleutels, URL's, provider-ID's, model-ID's en labels van plugins/kanalen blijven letterlijk ongewijzigd.

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # Expliciete Engelse overschrijving
```

## Niet-interactieve installatie

`--non-interactive` vereist `--accept-risk` (bevestigt dat agents krachtig zijn en dat volledige systeemtoegang riskant is). `--mode` is standaard `local`.

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai \
  --custom-image-input
```

`--custom-api-key` is optioneel; als deze wordt weggelaten, controleert onboarding `CUSTOM_API_KEY` in de omgeving. OpenClaw markeert gangbare ID's van visiemodellen (GPT-4o/4.1/5.x, Claude 3/4, Gemini, Qwen-VL, LLaVA, Pixtral en vergelijkbare modellen) automatisch als geschikt voor afbeeldingen. Geef `--custom-image-input` door voor onbekende aangepaste visie-ID's, of `--custom-text-input` om metadata voor alleen tekst af te dwingen. Gebruik `--custom-compatibility openai-responses` voor OpenAI-compatibele eindpunten die `/v1/responses` ondersteunen, maar niet `/v1/chat/completions`; geldige waarden zijn `openai` (standaard), `openai-responses`, `anthropic`.

LM Studio heeft ook een providerspecifieke sleutelvlag:

```bash
openclaw onboard --non-interactive \
  --auth-choice lmstudio \
  --custom-base-url "http://localhost:1234/v1" \
  --custom-model-id "qwen/qwen3.5-9b" \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --accept-risk
```

Niet-interactieve Ollama:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

`--custom-base-url` is standaard `http://127.0.0.1:11434`. `--custom-model-id` is optioneel; als deze wordt weggelaten, gebruikt onboarding de voorgestelde standaardwaarden van Ollama. ID's van cloudmodellen, zoals `kimi-k2.5:cloud`, werken hier ook.

Sla providersleutels op als verwijzingen in plaats van platte tekst:

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

Met `--secret-input-mode ref` schrijft onboarding door omgevingsvariabelen ondersteunde verwijzingen in plaats van sleutelwaarden als platte tekst: voor providers die door een authenticatieprofiel worden ondersteund, wordt `keyRef: { source: "env", provider: "default", id: <envVar> }` geschreven; voor aangepaste providers wordt `models.providers.<id>.apiKey` op dezelfde manier geschreven (bijvoorbeeld `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`). Contract: stel de omgevingsvariabele van de provider in de procesomgeving van onboarding in (bijvoorbeeld `OPENAI_API_KEY`) en geef niet ook een inline sleutelvlag door, tenzij die omgevingsvariabele is ingesteld. Een vlagwaarde zonder de bijbehorende omgevingsvariabele mislukt onmiddellijk met aanwijzingen.

### Gateway-authenticatie (niet-interactief)

- `--gateway-auth token --gateway-token <token>` slaat een token als platte tekst op. `token` is de standaardauthenticatiemodus.
- `--gateway-auth token --gateway-token-ref-env <name>` slaat `gateway.auth.token` op als een SecretRef naar een omgevingsvariabele. Vereist in de procesomgeving van onboarding een niet-lege omgevingsvariabele met die naam.
- `--gateway-token` en `--gateway-token-ref-env` sluiten elkaar wederzijds uit.
- Met `--install-daemon`: een door SecretRef beheerde `gateway.auth.token` wordt gevalideerd, maar niet als opgeloste platte tekst opgeslagen in de omgevingsmetadata van de supervisorservice; als de verwijzing niet kan worden opgelost, mislukt de installatie gesloten met aanwijzingen voor herstel. Als zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd en `gateway.auth.mode` niet is ingesteld, wordt de installatie geblokkeerd totdat de modus expliciet is ingesteld.
- Lokale onboarding schrijft `gateway.mode="local"` naar de configuratie. Als `gateway.mode` later in een configuratiebestand ontbreekt, duidt dit op beschadigde configuratie of een onvolledige handmatige bewerking, niet op een geldige snelkoppeling voor de lokale modus.
- Lokale onboarding installeert downloadbare plugins die voor het gekozen installatiepad vereist zijn (bijvoorbeeld een Codex- of Copilot-runtimeplugin voor die authenticatiekeuzes). Externe onboarding schrijft alleen verbindingsinformatie voor de externe Gateway; lokale pluginpakketten worden nooit geïnstalleerd.
- `--allow-unconfigured` is een afzonderlijke nooduitgang voor `openclaw gateway run`; onboarding kan hiermee `gateway.mode` niet overslaan.

```bash
export OPENAI_API_KEY="jouw-providersleutel"
export OPENCLAW_GATEWAY_TOKEN="jouw-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

### Status van lokale Gateway

- Tenzij je `--skip-health` doorgeeft, wacht onboarding op een bereikbare lokale Gateway voordat het succesvol wordt afgesloten.
- `--install-daemon` start eerst het installatiepad voor de beheerde Gateway. Zonder deze optie moet er al een lokale Gateway actief zijn (bijvoorbeeld `openclaw gateway run`).
- `--skip-health` slaat het wachten over als je in automatisering alleen configuratie-, werkruimte- en bootstrapgegevens wilt schrijven.
- `--skip-bootstrap` stelt `agents.defaults.skipBootstrap: true` in en slaat het aanmaken van `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` en `BOOTSTRAP.md` over.
- Op native Windows probeert `--install-daemon` eerst Scheduled Tasks en valt terug op een aanmeldingsitem per gebruiker in de Startup-map als het aanmaken van de taak wordt geweigerd.

### Interactieve verwijzingsmodus

- Kies **Geheime verwijzing gebruiken** wanneer daarom wordt gevraagd en kies vervolgens **Omgevingsvariabele** of een geconfigureerde geheimenprovider (`file` of `exec`).
- Onboarding voert een snelle preflightvalidatie uit voordat de verwijzing wordt opgeslagen en laat je het bij een fout opnieuw proberen.

### Keuzes voor Z.AI-eindpunten

<Note>
`--auth-choice zai-api-key` detecteert automatisch het beste Z.AI-eindpunt en -model voor je sleutel: Coding Plan-eindpunten geven de voorkeur aan `zai/glm-5.2` (met `glm-5.1` als terugvaloptie indien niet beschikbaar); algemene API-eindpunten gebruiken standaard `zai/glm-5.1`. Kies rechtstreeks `zai-coding-global` of `zai-coding-cn` om een Coding Plan-eindpunt af te dwingen.
</Note>

```bash
# Eindpuntselectie zonder prompt
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# Andere keuzes voor Z.AI-eindpunten: zai-coding-cn, zai-global, zai-cn
```

Mistral:

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

## Aanvullende niet-interactieve vlaggen

Modelauthenticatie op basis van tokens (gebruikt met `--auth-choice token`):

| Vlag                            | Beschrijving                                                                                                                 |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--token-provider <id>`         | ID van de tokenprovider die het token uitgeeft                                                                                         |
| `--token <token>`               | Tokenwaarde voor modelauthenticatie                                                                                        |
| `--token-profile-id <id>`       | ID van het authenticatieprofiel (standaard `<provider>:manual`; sommige flows die eigendom zijn van een provider gebruiken hun eigen standaardwaarde, zoals `anthropic:default`) |
| `--token-expires-in <duration>` | Optionele vervalduur van het token (bijvoorbeeld `365d`, `12h`)                                                                         |

Cloudflare AI Gateway: `--cloudflare-ai-gateway-account-id <id>`, `--cloudflare-ai-gateway-gateway-id <id>`.

Beheer van daemoninstallatie: `--no-install-daemon` / `--skip-daemon` (aliassen; sla installatie van de Gateway-service over), `--daemon-runtime <node>`.

Skills: `--node-manager <npm|pnpm|bun>` (standaard `npm`), `--skip-skills`.

Instellen van UI en hooks: `--skip-ui` (sla prompts voor Control UI/TUI over), `--skip-hooks` (sla het instellen van webhooks/hooks over), `--skip-channels`, `--skip-search`.

Uitvoer: `--suppress-gateway-token-output` onderdrukt Gateway-/UI-uitvoer die tokens bevat (tokenhints, een URL voor automatisch aanmelden met een ingesloten token en het automatisch starten van Control UI). Dit is nuttig in gedeelde terminals en CI.

<Note>
`--json` impliceert geen niet-interactieve modus bij begeleide of klassieke onboarding.
Met `--modern` is JSON een eenmalig OpenClaw-overzicht en wordt het programma na dat
ene resultaat afgesloten. Gebruik `--non-interactive` voor andere scripts.
</Note>

## Vooraf filteren van providers

Wanneer een authenticatiekeuze een voorkeursprovider impliceert, filtert onboarding de keuzelijsten voor het standaardmodel en de toestemmingslijst vooraf op de modellen van die provider. Het filter komt ook overeen met andere providers die eigendom zijn van dezelfde plugin, waaronder Coding Plan-varianten zoals `volcengine`/`volcengine-plan` en `byteplus`/`byteplus-plan`. Als het voorkeursproviderfilter geen geladen modellen oplevert, valt onboarding terug op de ongefilterde catalogus in plaats van de keuzelijst leeg te laten.

## Vervolgvragen voor zoeken op het web

Sommige providers voor zoeken op het web activeren tijdens onboarding providerspecifieke vervolgvragen:

- **Grok** kan optionele configuratie van `x_search` aanbieden met dezelfde xAI-authenticatie en een modelkeuze voor `x_search`.
- **Kimi** kan vragen naar de Moonshot API-regio (`api.moonshot.ai` tegenover `api.moonshot.cn`) en het standaardmodel van Kimi voor zoeken op het web.

## Ander gedrag

- Gedrag van het DM-bereik bij lokale onboarding: [Naslaginformatie voor CLI-installatie](/nl/start/wizard-cli-reference#outputs-and-internals).
- Snelste eerste chat: `openclaw dashboard` (Control UI, zonder kanaalinstallatie).
- Aangepaste provider: verbind elk OpenAI- of Anthropic-compatibel eindpunt, inclusief gehoste providers die niet in de lijst staan. Gebruik compatibiliteit **Onbekend** voor automatische detectie via een live-probe.
- Als een Hermes-status wordt gedetecteerd, biedt onboarding een migratieflow aan (zie `--flow import` hierboven).

## Veelgebruikte vervolgopdrachten

Gebruik `openclaw configure` later voor gerichte wijzigingen zonder inferentie en `openclaw
channels add` voor het instellen van alleen kanalen. Voer voor wijzigingen aan de modelprovider of authenticatieroute
in plaats daarvan `openclaw onboard` uit.

```bash
openclaw channels add
openclaw configure
openclaw agents add <name>
```
