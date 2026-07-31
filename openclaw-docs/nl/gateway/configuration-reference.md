---
read_when:
    - Je hebt exacte configuratiebetekenissen of standaardwaarden op veldniveau nodig
    - Je valideert configuratieblokken voor kanalen, modellen, de Gateway of tools
summary: Gateway-configuratiereferentie voor OpenClaw-basissleutels, standaardwaarden en links naar specifieke subsysteemreferenties
title: Configuratiereferentie
x-i18n:
    generated_at: "2026-07-27T05:10:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

Referentie op veldniveau voor `~/.openclaw/openclaw.json`: sleutels, standaardwaarden en links naar uitgebreidere subsysteempagina's. Zie [Configuratie](/nl/gateway/configuration) voor taakgerichte installatie-instructies. Opdrachtenoverzichten die bij kanalen en plugins horen en geavanceerde instellingen voor geheugen/QMD staan op hun eigen pagina's, niet hier.

De configuratie-indeling is **JSON5** (opmerkingen + afsluitende komma's toegestaan). Alle velden zijn optioneel; OpenClaw gebruikt veilige standaardwaarden wanneer ze worden weggelaten.

De code is leidend boven deze pagina:

- `openclaw config schema` geeft het actuele JSON Schema weer dat wordt gebruikt voor validatie en de Control UI, samengevoegd met metadata van gebundelde plugins en kanalen.
- Agents moeten de toolactie `config.schema.lookup` van de tool `gateway` aanroepen voor één exact, tot een pad beperkt schemaknooppunt voordat ze de configuratie bewerken.
- `pnpm config:docs:check` / `pnpm config:docs:gen` valideren de baseline-hash van dit document aan de hand van het huidige schemaoppervlak.

Schema-`uiHints` bevatten ook een opgeloste Booleaanse waarde `advanced` voor elk pad.
De Control UI gebruikt deze om algemene velden als eerste weer te geven en geavanceerde velden per
sectie in te klappen; zoeken omvat nog steeds beide niveaus. Niveaumetadata is uitsluitend voor de presentatie.
Wanneer je een sleutel toevoegt, declareer je het niveau ervan op het blad of laat je dit overnemen van de dichtstbijzijnde
bovenliggende declaratie. Een pad zonder gedeclareerde bovenliggende waarde is standaard geavanceerd.

Specifieke uitgebreide referenties:

- [Referentie voor geheugenconfiguratie](/nl/reference/memory-config) voor `memory.search.*`, `memory.qmd.*`, `memory.citations` en Dreaming-configuratie onder `plugins.entries.memory-core.config.dreaming`.
- [Slash-opdrachten](/nl/tools/slash-commands) voor het huidige overzicht van ingebouwde + gebundelde opdrachten.
- Pagina's van de verantwoordelijke kanalen/plugins voor kanaalspecifieke opdrachtoppervlakken.

---

## Kanalen

Configuratiesleutels per kanaal staan in [Configuratie - kanalen](/nl/gateway/config-channels): `channels.*` voor Slack, Discord, Telegram, WhatsApp, Matrix, iMessage en andere gebundelde kanalen (authenticatie, toegangsbeheer, meerdere accounts, vermeldingstoegang).

## Standaardwaarden voor agents, meerdere agents, sessies en berichten

Zie [Configuratie - agents](/nl/gateway/config-agents) voor:

- `agents.defaults.*` (werkruimte, model, redeneren, Heartbeat, geheugen, media, Skills, sandbox)
- `multiAgent.*` (routering en bindingen voor meerdere agents)
- `session.*` (sessielevenscyclus, Compaction, opschoning)
- `messages.*` (berichtbezorging, TTS, Markdown-weergave)
- `talk.*` (spraakmodus)
  - `talk.consultThinkingLevel`: overschrijving van het redeneerniveau voor de volledige OpenClaw-agentuitvoering achter realtime raadplegingen via Spraak in de Control UI
  - `talk.consultFastMode`: eenmalige overschrijving voor snelle modus voor realtime raadplegingen via Spraak in de Control UI
  - `talk.speechLocale`: optionele BCP 47-landinstellings-id voor spraakherkenning van Spraak op Android, iOS en macOS
  - `talk.silenceTimeoutMs`: wanneer dit niet is ingesteld, behoudt Spraak het standaard pauzevenster van het platform voordat het transcript wordt verzonden (`700 ms on macOS and Android, 900 ms on iOS`)
  - `talk.realtime.consultRouting`: Gateway-relayterugval voor voltooide realtime Spraak-transcripten die `openclaw_agent_consult` overslaan

## Tools en aangepaste providers

Toolbeleid, experimentele schakelaars, door providers ondersteunde toolconfiguratie en instellingen voor aangepaste
providers/basis-URL's staan in
[Configuratie - tools en aangepaste providers](/nl/gateway/config-tools).

## Modellen

Providerdefinities, lijsten met toegestane modellen en instellingen voor aangepaste providers staan in
[Configuratie - tools en aangepaste providers](/nl/gateway/config-tools#custom-providers-and-base-urls).
De hoofdsectie `models` beheert ook het algemene gedrag van de modelcatalogus.

```json5
{
  models: {
    // Optioneel. Standaard: true. Vereist een herstart van de Gateway wanneer dit wordt gewijzigd.
    pricing: { enabled: false },
  },
}
```

- `models.mode`: gedrag van de providercatalogus (`merge` of `replace`).
- `models.providers`: aangepaste providertoewijzing met provider-id als sleutel.
- `models.providers.*.localService`: optionele procesbeheerder op aanvraag voor
  lokale modelservers. OpenClaw controleert het geconfigureerde gezondheidseindpunt, start
  zo nodig het absolute `command`, wacht tot het gereed is en verzendt vervolgens het modelverzoek.
  Zie [Lokale modelservices](/nl/gateway/local-model-services).
- `models.pricing.enabled`: beheert de prijsinitialisatie op de achtergrond die
  start nadat sidecars en kanalen het gereedheidspad van de Gateway bereiken. Wanneer `false`,
  slaat de Gateway het ophalen van prijscatalogi van OpenRouter en LiteLLM over; geconfigureerde
  `models.providers.*.models[].cost`-waarden blijven werken voor lokale kostenramingen.

## MCP

Door OpenClaw beheerde MCP-serverdefinities staan onder `mcp.servers` en worden
gebruikt door ingebedde OpenClaw- en andere runtime-adapters. De opdrachten `openclaw mcp list`,
`show`, `set` en `unset` beheren dit blok zonder tijdens configuratiebewerkingen verbinding te maken met de
doelserver.

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // Optionele projectiebesturing voor Codex app-server.
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`: benoemde stdio- of externe MCP-serverdefinities voor runtimes die
  geconfigureerde MCP-tools beschikbaar stellen.
  Externe vermeldingen gebruiken `transport: "streamable-http"` of `transport: "sse"`;
  `type: "http"` is een CLI-eigen alias die `openclaw mcp set` en
  `openclaw doctor --fix` normaliseren naar het canonieke veld `transport`.
- `mcp.servers.<name>.enabled`: stel `false` in om een opgeslagen serverdefinitie te behouden
  en deze tegelijkertijd uit te sluiten van ingebedde MCP-detectie en toolprojectie door OpenClaw.
- `mcp.servers.<name>.requestTimeoutMs`: MCP-verzoektime-out per server in milliseconden.
- `mcp.servers.<name>.connectionTimeoutMs`: verbindingstime-out per server in milliseconden.
- `mcp.servers.<name>.supportsParallelToolCalls`: optionele gelijktijdigheidsaanwijzing voor
  adapters die kunnen kiezen of ze parallelle MCP-toolaanroepen uitvoeren.
- `mcp.servers.<name>.auth`: stel `"oauth"` in voor HTTP-MCP-servers die
  OAuth vereisen. Voer `openclaw mcp login <name>` uit om tokens in de OpenClaw-status op te slaan.
- `mcp.servers.<name>.oauth`: optionele overschrijvingen voor OAuth-bereik, omleidings-URL en URL voor
  clientmetadata.
- `mcp.servers.<name>.sslVerify`, `clientCert`, `clientKey`: HTTP-TLS-instellingen
  voor privé-eindpunten en wederzijdse TLS.
- `mcp.servers.<name>.toolFilter`: optionele toolselectie per server. `include`
  beperkt de gedetecteerde MCP-tools tot overeenkomende namen; `exclude` verbergt overeenkomende
  namen. Vermeldingen zijn exacte namen van MCP-tools of eenvoudige `*`-globpatronen. Servers met
  resources of prompts genereren ook namen voor hulptools (`resources_list`,
  `resources_read`, `prompts_list`, `prompts_get`), waarop hetzelfde
  filter wordt toegepast.
- `mcp.servers.<name>.codex`: optionele projectiebesturing voor Codex app-server.
  Dit blok is OpenClaw-metadata die uitsluitend geldt voor Codex app-server-threads; het heeft geen
  invloed op ACP-sessies, algemene Codex-harnasconfiguratie of andere runtime-adapters.
  Een niet-lege `codex.agents` beperkt de server tot de vermelde OpenClaw-agent-id's.
  Lege, blanco of ongeldige lijsten van agents binnen het bereik worden door de configuratievalidatie geweigerd
  en door het runtime-projectiepad weggelaten in plaats van algemeen te worden.
  `codex.defaultToolsApprovalMode` genereert de systeemeigen
  `default_tools_approval_mode` van Codex voor die server. OpenClaw verwijdert het blok `codex`
  voordat de systeemeigen `mcp_servers`-configuratie aan Codex wordt doorgegeven. Laat het blok weg om
  de server geprojecteerd te houden voor elke Codex app-server-agent met het standaard
  MCP-goedkeuringsgedrag van Codex.
- Gebundelde MCP-runtimes met sessiebereik gebruiken een ingebouwde inactieve TTL van 10 minuten.
  Eenmalige ingebedde uitvoeringen vragen om opschoning aan het einde van de uitvoering; de TTL is het vangnet voor langlopende sessies en toekomstige aanroepers.
- Wijzigingen onder `mcp.*` worden direct toegepast door MCP-runtimes van gecachte sessies te verwijderen.
  Bij de volgende tooldetectie of het volgende toolgebruik worden ze opnieuw aangemaakt op basis van de nieuwe configuratie, zodat verwijderde
  `mcp.servers`-vermeldingen onmiddellijk worden opgeruimd in plaats van te wachten op de inactieve TTL.
- Runtimedetectie houdt ook rekening met meldingen over wijzigingen in de lijst met MCP-tools door
  de gecachte catalogus voor die sessie te verwijderen. Servers die resources of
  prompts adverteren, krijgen hulptools voor het weergeven/lezen van resources en het weergeven/ophalen van
  prompts. Bij herhaalde mislukte toolaanroepen wordt de betreffende server kort gepauzeerd voordat
  een nieuwe aanroep wordt geprobeerd.

Zie [MCP](/nl/cli/mcp#openclaw-as-an-mcp-client-registry) en
[CLI-backends](/nl/gateway/cli-backends#bundle-mcp-overlays) voor runtimegedrag.

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // of tekenreeks met platte tekst
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: optionele lijst met toegestane items, uitsluitend voor gebundelde Skills (beheerde Skills en Skills in de werkruimte worden niet beïnvloed).
- `load.extraDirs`: aanvullende gedeelde hoofdmappen voor Skills (laagste prioriteit).
- `load.allowSymlinkTargets`: vertrouwde echte doelhoofdmappen waarnaar symlinks van Skills mogen
  verwijzen wanneer de koppeling zich buiten de geconfigureerde bronhoofdmap bevindt.
- `workshop.allowSymlinkTargetWrites`: staat toe dat toepassen via Skill Workshop schrijft
  via reeds vertrouwde symlinkdoelen (standaard: false).
- `install.preferBrew`: indien true, wordt de voorkeur gegeven aan Homebrew-installatieprogramma's wanneer `brew`
  beschikbaar is, voordat wordt teruggevallen op andere soorten installatieprogramma's.
- `install.nodeManager`: voorkeur voor het Node-installatieprogramma voor `metadata.openclaw.install`-
  specificaties (`npm` | `pnpm` | `yarn` | `bun`).
- `install.allowUploadedArchives`: sta vertrouwde `operator.admin`-Gateway-
  clients toe om privé-ziparchieven te installeren die via `skills.upload.*` zijn klaargezet
  (standaard: false). Dit schakelt alleen het pad voor geüploade archieven in; normale ClawHub-
  installaties vereisen dit niet.
- `entries.<skillKey>.enabled: false` schakelt een Skill uit, zelfs als deze gebundeld/geïnstalleerd is.
- `entries.<skillKey>.apiKey`: gemaksoptie voor Skills die een primaire omgevingsvariabele declareren (tekenreeks met platte tekst of SecretRef-object).
- `limits.maxCandidatesPerRoot`, `limits.maxSkillsLoadedPerSource`, `limits.maxSkillsInPrompt`, `limits.maxSkillsPromptChars`, `limits.maxSkillFileBytes`: begrenzen de detectie van Skills en de op het model gerichte Skills-prompt.
- Instellingen voor autonomie/goedkeuring van Skill Workshop (`workshop.autonomous.enabled`, `workshop.approvalPolicy`, `workshop.maxPending`, `workshop.maxSkillBytes`) worden beschreven in [Skills-configuratie](/nl/tools/skills-config).

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- Geladen uit pakket- of bundelmappen onder `~/.openclaw/extensions` en `<workspace>/.openclaw/extensions`, plus bestanden of mappen die zijn vermeld in `plugins.load.paths`.
- Plaats zelfstandige pluginbestanden in `plugins.load.paths`; automatisch ontdekte extensiehoofdmappen negeren `.js`-, `.mjs`- en `.ts`-bestanden op het hoogste niveau, zodat hulpscripts in die hoofdmappen het opstarten niet blokkeren.
- Detectie accepteert native OpenClaw-plugins, compatibele Codex-bundels en Claude-bundels, waaronder Claude-bundels zonder manifest met de standaardindeling.
- **Voor configuratiewijzigingen moet de Gateway opnieuw worden gestart.**
- `allow`: optionele toelatingslijst (alleen vermelde plugins worden geladen). `deny` heeft voorrang.
- `plugins.entries.<id>.apiKey`: handig veld voor een API-sleutel op pluginniveau (wanneer ondersteund door de plugin).
- `plugins.entries.<id>.env`: aan de plugin gekoppelde toewijzing van omgevingsvariabelen.
- `plugins.entries.<id>.hooks.allowPromptInjection`: wanneer `false`, blokkeert de kern hooks die prompts wijzigen, zoals `before_prompt_build`. Dit geldt voor native pluginhooks en ondersteunde hookmappen die door bundels worden geleverd.
- `plugins.entries.<id>.hooks.allowConversationAccess`: wanneer `true`, mogen vertrouwde niet-gebundelde plugins onbewerkte gespreksinhoud lezen via getypeerde hooks zoals `llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize` en `agent_end`.
- `plugins.entries.<id>.subagent.allowModelOverride`: vertrouw deze plugin expliciet om per uitvoering overschrijvingen voor `provider` en `model` aan te vragen voor subagentuitvoeringen op de achtergrond.
- `plugins.entries.<id>.subagent.allowedModels`: optionele toelatingslijst met canonieke `provider/model`-doelen voor vertrouwde subagentoverschrijvingen. Gebruik `"*"` alleen wanneer je bewust elk model wilt toestaan.
- `plugins.entries.<id>.llm.allowModelOverride`: vertrouw deze plugin expliciet om modeloverschrijvingen voor `api.runtime.llm.complete` aan te vragen.
- `plugins.entries.<id>.llm.allowedModels`: optionele toelatingslijst met canonieke `provider/model`-doelen voor vertrouwde overschrijvingen van LLM-aanvullingen door plugins. Gebruik `"*"` alleen wanneer je bewust elk model wilt toestaan.
- `plugins.entries.<id>.llm.allowAgentIdOverride`: vertrouw deze plugin expliciet om `api.runtime.llm.complete` uit te voeren voor een niet-standaard agent-ID.
- `plugins.entries.<id>.config`: door de plugin gedefinieerd configuratieobject (gevalideerd aan de hand van het native OpenClaw-pluginschema indien beschikbaar).
- Account- en runtime-instellingen van kanaalplugins staan onder `channels.<id>` en moeten worden beschreven door de `channelConfigs`-metadata van het manifest van de beherende plugin, niet door een centraal OpenClaw-optieregister.

### Configuratie van de Codex-harnasplugin

De gebundelde plugin `codex` beheert de instellingen van het native Codex-appserverharnas onder
`plugins.entries.codex.config`. Zie de
[referentie voor het Codex-harnas](/nl/plugins/codex-harness-reference) voor het volledige configuratieoppervlak
en [Codex-harnas](/nl/plugins/codex-harness) voor het runtimemodel.

`codexPlugins` geldt alleen voor sessies die het native Codex-harnas selecteren.
Hiermee worden Codex-plugins niet ingeschakeld voor OpenClaw-provideruitvoeringen, ACP-
gesprekskoppelingen of een ander harnas dan Codex.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: schakelt native ondersteuning voor Codex-
  plugins/apps in voor het Codex-harnas. Standaard: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`: stelt elke
  momenteel toegankelijke app die met het geauthenticeerde Codex-account is verbonden beschikbaar in
  elke nieuwe native Codex-thread. Standaard: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  standaardbeleid voor destructieve acties bij geconfigureerde plugin-appverzoeken.
  Gebruik `true` om veilige Codex-goedkeuringsschema's zonder prompt te accepteren, `false`
  om ze af te wijzen, `"auto"` om door Codex vereiste goedkeuringen via OpenClaw-
  plugingoedkeuringen te leiden, of `"ask"` om bij elke schrijvende/destructieve
  pluginactie om bevestiging te vragen zonder permanente goedkeuring. De modus `"ask"` wist permanente Codex-
  goedkeuringsoverschrijvingen per tool voor de betreffende app en selecteert de menselijke
  goedkeuringsbeoordelaar voor die app voordat de Codex-thread begint.
  Standaard: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: schakelt een
  geconfigureerde pluginvermelding in wanneer de globale instelling `codexPlugins.enabled` ook waar is.
  Standaard: `true` voor expliciete vermeldingen.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`:
  stabiele marketplace-identiteit, vereist samen met `pluginName` voor elke opgeloste
  vermelding. Ondersteunt `"openai-curated"` en `"workspace-directory"`. Vermeldingen
  waarbij een van beide identiteitsvelden ontbreekt, worden genegeerd.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`: stabiele
  Codex-pluginidentiteit, vereist samen met `marketplaceName`. Een
  `workspace-directory`-vermelding moet exact de door de marketplace gekwalificeerde
  `summary.id` gebruiken die door `plugin/list` wordt geretourneerd, bijvoorbeeld
  `"example-plugin@workspace-directory"`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`:
  overschrijving van destructieve acties per plugin. Wanneer deze ontbreekt, wordt de globale
  waarde `allow_destructive_actions` gebruikt. De waarde per plugin accepteert hetzelfde
  beleid `true`, `false`, `"auto"` of `"ask"`.

Elke toegelaten plugin-app die `"ask"` gebruikt, stuurt de goedkeuringsverzoeken van die app
naar de menselijke beoordelaar. Andere apps en goedkeuringen voor threads die geen app betreffen, behouden hun
geconfigureerde beoordelaar, zodat gemengd pluginbeleid het gedrag van `"ask"` niet overneemt.

`codexPlugins.enabled` is de globale inschakelingsrichtlijn. Expliciete plugin-
vermeldingen die door migratie zijn geschreven, vormen de permanente samengestelde set voor installatie- en herstelgeschiktheid.
Handmatig geconfigureerde `workspace-directory`-vermeldingen moeten al
zijn geïnstalleerd en ingeschakeld, en de apps die ze beheren moeten toegankelijk zijn; OpenClaw
installeert of authenticeert ze niet. Als Codex de expliciete aanvraag voor de werkruimtecatalogus
afwijst, worden ingeschakelde werkruimtevermeldingen gesloten geweigerd met
`marketplace_missing`, terwijl samengestelde vermeldingen uit de standaardcatalogus
beschikbaar blijven. `plugins["*"]` wordt niet ondersteund, er is geen `install`-schakelaar en
lokale `marketplacePath`-waarden zijn bewust geen configuratievelden, omdat ze
hostspecifiek zijn. Zie
[Native Codex-plugins](/nl/plugins/codex-native-plugins) voor vereisten voor de appserverversie en
gereedheid.

Gereedheidscontroles voor `app/list` worden één uur in de cache bewaard en
asynchroon vernieuwd wanneer ze verouderd zijn. De appconfiguratie voor Codex-threads wordt berekend bij het
opzetten van een Codex-harnassessie, niet bij elke beurt; gebruik `/new`, `/reset` of start de Gateway
opnieuw nadat je de configuratie van native plugins hebt gewijzigd.

`codexPlugins.allow_all_plugins` neemt een momentopname van elke momenteel toegankelijke account-
app op in elke nieuwe native Codex-thread. Het installeert geen plugins of apps, en
ontoegankelijke apps blijven uitgesloten. Account-apps gebruiken het globale
beleid `codexPlugins.allow_destructive_actions`. Expliciete pluginvermeldingen hebben
voorrang wanneer dezelfde app via beide paden aanwezig is. Als `app/list` niet kan worden
gelezen, wordt accountbrede beschikbaarstelling gesloten geweigerd.

- `plugins.entries.firecrawl.config.webFetch`: instellingen voor de Firecrawl-provider voor het ophalen van webinhoud.
  - `apiKey`: optionele Firecrawl-API-sleutel voor hogere limieten (accepteert SecretRef). Valt terug op de omgevingsvariabele `plugins.entries.firecrawl.config.webSearch.apiKey` of `FIRECRAWL_API_KEY`.
  - `baseUrl`: basis-URL van de Firecrawl-API (standaard: `https://api.firecrawl.dev`; zelfgehoste overschrijvingen moeten naar privé-/interne eindpunten verwijzen).
  - `onlyMainContent`: extraheer alleen de hoofdinhoud van pagina's (standaard: `true`).
  - `maxAgeMs`: maximale cacheleeftijd in milliseconden (standaard: `172800000` / 2 dagen).
  - `timeoutSeconds`: time-out voor scrapeverzoeken in seconden (standaard: `60`).
- `plugins.entries.xai.config.xSearch`: instellingen voor xAI X Search (Grok-webzoeken).
  - `enabled`: schakel de X Search-provider in.
  - `model`: Grok-model dat voor zoeken moet worden gebruikt (bijv. `"grok-4.3"`).
- `plugins.entries.memory-core.config.dreaming`: instellingen voor Dreaming van het geheugen. Zie [Dreaming](/nl/concepts/dreaming) voor fasen en drempelwaarden.
  - `enabled`: hoofdschakelaar voor Dreaming (standaard `false`).
  - `frequency`: Cron-interval voor elke volledige Dreaming-cyclus (standaard `"0 3 * * *"`).
  - `model`: optionele modeloverschrijving voor de Dream Diary-subagent. Vereist `plugins.entries.memory-core.subagent.allowModelOverride: true`; combineer met `allowedModels` om doelen te beperken. Fouten wegens een niet-beschikbaar model worden eenmaal opnieuw geprobeerd met het standaardsessiemodel; fouten met vertrouwen of toelatingslijsten vallen niet stilzwijgend terug.
  - Fasebeleid en drempelwaarden zijn implementatiedetails (geen configuratiesleutels voor gebruikers).
- De volledige geheugenconfiguratie staat in de [referentie voor geheugenconfiguratie](/nl/reference/memory-config):
  - `memory.search.*`
  - `agents.entries.*.memory.search.*` voor overschrijvingen per agent
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- Ingeschakelde Claude-bundelplugins kunnen ook ingesloten OpenClaw-standaardwaarden uit `settings.json` bijdragen; OpenClaw past deze toe als opgeschoonde agentinstellingen, niet als onbewerkte patches voor de OpenClaw-configuratie.
- `plugins.slots.memory`: kies de ID van de actieve geheugenplugin, of `"none"` om geheugenplugins uit te schakelen.
- `plugins.slots.contextEngine`: kies de ID van de actieve contextengineplugin; standaard is dit `"legacy"`, tenzij je een andere engine installeert en selecteert.

Zie [Plugins](/nl/tools/plugin).

---

## Browser

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // opt in only for trusted private-network access
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` schakelt `act:evaluate` en `wait --fn` uit.
- `tabCleanup` regelt periodieke opschoning op basis van beste inspanning voor bijgehouden tabbladen van de primaire agent
  na inactiviteit of wanneer een sessie de limiet overschrijdt. Alleen tabbladen
  die zijn gemaakt door de browsertool `action: "open"` worden bijgehouden; tabbladen die door de gebruiker zijn geopend of
  waarvan het eigenaarschap onbekend is, worden nooit overgenomen. Het uitschakelen van `tabCleanup` schakelt expliciete opschoning van de sessielevenscyclus niet uit.
- Hostlokale openingen met een stabiel native CDP-doel en een stabiele browseridentiteit worden
  opgeslagen in de gedeelde SQLite-status en blijven na herstarts van de Gateway in aanmerking komen voor
  `/new` en opschoning van de sessielevenscyclus. Native CDP-doelen voor tools
  blijven na een herstart ook in aanmerking komen voor opschoning bij inactiviteit en overschrijding van de limiet. Chrome MCP gebruikt
  proceslokale doelhandles, zodat koude records van bestaande sessies wachten op
  opschoning van de levenscyclus, in plaats van het risico te lopen dat een inactiviteitsopruiming wordt uitgevoerd op niet-toewijsbare
  activiteit na een herstart. OpenClaw verifieert het profiel en de browserinstantie
  voordat deze worden gesloten. Automatisch verbinden van Chrome MCP, een ontbrekende
  browseridentiteit voor `/json/version` en niet-opgeloste native doelen blijven volledig proceslokaal,
  zodat ze na een herstart niet automatisch worden gesloten. Oudere, niet-bijgehouden tabbladen moeten
  handmatig worden gesloten. Tijdelijke fouten blijven in behandeling voor een latere nieuwe poging. Zie
  [Eigenaarschap van tabbladopschoning](/nl/tools/browser#tab-cleanup-ownership).
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` is uitgeschakeld wanneer deze niet is ingesteld, zodat browsernavigatie standaard strikt blijft.
- Stel `ssrfPolicy.dangerouslyAllowPrivateNetwork: true` alleen in wanneer je browsernavigatie naar privénetwerken bewust vertrouwt.
- In de strikte modus geldt voor externe CDP-profieleindpunten (`profiles.*.cdpUrl`) dezelfde blokkering van privénetwerken tijdens bereikbaarheids- en detectiecontroles.
- `ssrfPolicy.allowPrivateNetwork` blijft ondersteund als verouderde alias.
- Gebruik in de strikte modus `ssrfPolicy.hostnameAllowlist` en `ssrfPolicy.allowedHostnames` voor expliciete uitzonderingen.
- Externe profielen kunnen alleen worden gekoppeld (starten/stoppen/resetten uitgeschakeld).
- `profiles.*.cdpUrl` accepteert `http://`, `https://`, `ws://` en `wss://`.
  Gebruik HTTP(S) wanneer OpenClaw `/json/version` moet detecteren; gebruik WS(S)
  wanneer je provider je een directe DevTools-WebSocket-URL geeft.
- Als een extern beheerde CDP-service via loopback bereikbaar is, stel dan
  `attachOnly: true` van dat profiel in; anders behandelt OpenClaw de loopbackpoort als een
  lokaal beheerd browserprofiel en kan het fouten over het eigenaarschap van de lokale poort melden.
- `existing-session`-profielen gebruiken Chrome MCP in plaats van CDP en kunnen worden gekoppeld op
  de geselecteerde host of via een verbonden browsernode.
- `existing-session`-profielen kunnen `userDataDir` instellen om een specifiek
  Chromium-gebaseerd browserprofiel te gebruiken, zoals Brave of Edge.
- `existing-session`-profielen kunnen `cdpUrl` instellen wanneer Chrome al actief is
  achter een DevTools HTTP(S)-detectie-eindpunt of direct WS(S)-eindpunt. In die
  modus geeft OpenClaw het eindpunt door aan Chrome MCP in plaats van automatisch verbinden te gebruiken;
  `userDataDir` wordt genegeerd voor de startargumenten van Chrome MCP.
- `existing-session`-profielen behouden de huidige routelimieten van Chrome MCP:
  acties op basis van momentopnamen/verwijzingen in plaats van doelen via CSS-selectors, uploadhooks
  voor één bestand, geen overschrijvingen van dialoogtime-outs, geen `wait --load networkidle` en geen
  `responsebody`, PDF-export, onderschepping van downloads of batchacties.
- Lokaal beheerde `openclaw`-profielen wijzen automatisch `cdpPort` en `cdpUrl` toe; stel
  `cdpUrl` alleen expliciet in voor externe CDP-profielen of koppeling aan een
  eindpunt van een bestaande sessie.
- Lokaal beheerde profielen kunnen `executablePath` instellen om de globale
  `browser.executablePath` voor dat profiel te overschrijven. Gebruik dit om het ene profiel in
  Chrome en het andere in Brave uit te voeren.
- Volgorde van automatische detectie: standaardbrowser indien gebaseerd op Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- `browser.executablePath` en `browser.profiles.<name>.executablePath`
  accepteren beide `~` en `~/...` voor de thuismap van je besturingssysteem vóór het starten van Chromium.
  `userDataDir` per profiel op `existing-session`-profielen wordt ook uitgebreid vanuit de tilde.
- Besturingsservice: alleen loopback (poort afgeleid van `gateway.port`, standaard `18791`).
- `extraArgs` voegt extra startvlaggen toe aan het lokaal starten van Chromium (bijvoorbeeld
  `--disable-gpu`, vensterafmetingen of debugvlaggen).

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji, korte tekst, afbeeldings-URL of data-URI
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // Behoud commentaar na uitvoeringen in de Control UI; levert het niet aan kanalen
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue; weglaten om de wachtrijmodus van de server te gebruiken
      showAdvancedSettings: false, // Vouw elke groep Advanced in Settings uit
    },
  },
}
```

- `seamColor`: accentkleur voor de UI-elementen van native apps (tint van de Talk Mode-ballon, enzovoort).
- `assistant`: overschrijving van de Control UI-identiteit. Valt terug op de identiteit van de actieve agent.
- `prefs`: apparaatoverschrijdende operatorvoorkeuren. Dit is de canonieke locatie, zodat agents
  deze via de goedkeuringspoort kunnen wijzigen en elke Control UI-client
  gesynchroniseerd blijft; browsers spiegelen de waarden naar lokale opslag voor direct opstarten en bewaren
  een apparaatlokale kopie wanneer ze de configuratie niet kunnen schrijven (viewerscope, offline).
  `chatPersistCommentary` is standaard `true`. Als je dit instelt op `false`, blijft live
  commentaar tijdens een uitvoering zichtbaar, maar wordt het na voltooiing verwijderd en wordt voorkomen dat nieuw
  Codex-commentaar in de duurzame transcriptspiegel terechtkomt. Levering via berichtkanalen
  blijft afzonderlijk en ongewijzigd.
  `showAdvancedSettings` is standaard `false`; zoeken in Settings kan tijdelijk
  één overeenkomende geavanceerde groep openen zonder deze voorkeur te wijzigen.
  Voorkeuren die alleen de presentatie beïnvloeden, zoals tekstschaal, chatbreedte en live
  activiteit in de zijbalk, blijven browserlokaal en worden geconfigureerd in Settings.
  Verbonden clients passen wijzigingen aan de serverzijde direct toe: de Gateway verzendt
  na elke opgeslagen configuratieschrijfactie een `config.changed`-gebeurtenis met alleen een hash, waarna
  clients hun momentopname vernieuwen (overgeslagen zolang een lokaal instellingenconcept
  niet-opgeslagen wijzigingen bevat). Clients die opnieuw verbinding maken, synchroniseren bij het verbinden.

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // of OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // voor mode=trusted-proxy; zie /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // optionele AI-doeltitels voor toolaanroepen (verbruikt tokens van het hulpprogrammamodel)
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // gevaarlijk: absolute externe http(s)-URL's voor insluiting toestaan
      // allowedOrigins: ["https://control.example.com"], // vereist voor een Control UI buiten loopback
      // dangerouslyAllowHostHeaderOriginFallback: false, // gevaarlijke terugvalmodus voor oorsprong via de Host-header
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // Optioneel. Standaard false.
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // Optioneel. Standaard niet ingesteld/uitgeschakeld.
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // Automatische goedkeuring met SSH-verificatie. Standaard: ingeschakeld (true).
        // Stel false in om alleen SSH-verificatie uit te schakelen; dit heeft geen invloed op
        // autoApproveCidrs hierboven. Stel voor uitsluitend handmatige nodekoppeling false in EN
        // verwijder autoApproveCidrs. Geef een object door om af te stemmen: { user, identity,
        // timeoutMs, cidrs }.
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // Aanvullende HTTP-weigeringen voor /tools/invoke
      deny: ["browser"],
      // Verwijder tools uit de standaard HTTP-weigeringslijst voor aanroepen door eigenaren/beheerders
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Details van Gateway-velden">

- `mode`: `local` (Gateway uitvoeren) of `remote` (verbinding maken met externe Gateway). Gateway weigert te starten tenzij `local`.
- `port`: één gemultiplexte poort voor WS + HTTP. Prioriteit: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto`, `loopback` (standaard), `lan` (`0.0.0.0`), `tailnet` (Tailscale-IPv4 indien beschikbaar, anders loopback), of `custom` (één IPv4-adres). Een herleid `tailnet`-adres en elk `custom`-adres anders dan `127.0.0.1` of `0.0.0.0` vereisen `127.0.0.1` op dezelfde poort voor clients op dezelfde host; het opstarten mislukt als een van beide listeners niet kan worden gebonden. Blootstelling buiten loopback blijft beperkt tot de geselecteerde interface.
- **Verouderde bind-aliassen**: gebruik bindmoduswaarden in `gateway.bind` (`auto`, `loopback`, `lan`, `tailnet`, `custom`), geen hostaliassen (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`).
- **Docker-opmerking**: de standaard `loopback`-binding luistert binnen de container op `127.0.0.1`. Met Docker-bridgenetwerken (`-p 18789:18789`) komt verkeer binnen op `eth0`, waardoor de Gateway onbereikbaar is. Gebruik `--network host`, of stel `bind: "lan"` (of `bind: "custom"` met `customBindHost: "0.0.0.0"`) in om op alle interfaces te luisteren.
- **Authenticatie**: standaard vereist. Bindingen buiten loopback vereisen Gateway-authenticatie. In de praktijk betekent dit een gedeeld token/wachtwoord of een identiteitsbewuste reverse proxy met `gateway.auth.mode: "trusted-proxy"`. De onboardingwizard genereert standaard een token.
- Als zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd (inclusief SecretRefs), stel `gateway.auth.mode` dan expliciet in op `token` of `password`. Opstart- en service-installatie-/reparatieprocessen mislukken wanneer beide zijn geconfigureerd en de modus niet is ingesteld.
- `gateway.auth.mode: "none"`: expliciete modus zonder authenticatie. Gebruik deze alleen voor vertrouwde lokale loopbackconfiguraties; deze optie wordt bewust niet aangeboden in onboardingprompts.
- `gateway.auth.mode: "trusted-proxy"`: delegeer browser-/gebruikersauthenticatie aan een identiteitsbewuste reverse proxy en vertrouw identiteitsheaders van `gateway.trustedProxies` (zie [Vertrouwde proxy-authenticatie](/nl/gateway/trusted-proxy-auth)). Deze modus verwacht standaard een proxybron **buiten loopback**; loopback-reverse-proxy's op dezelfde host vereisen expliciet `gateway.auth.trustedProxy.allowLoopback = true`. Interne aanroepers op dezelfde host kunnen `gateway.auth.password` gebruiken als lokale directe terugvaloptie; `gateway.auth.token` blijft wederzijds uitsluitend met de vertrouwde-proxymodus.
- `gateway.auth.allowTailscale`: wanneer `true`, kunnen Tailscale Serve-identiteitsheaders voldoen aan de authenticatie voor de Control UI/WebSocket (geverifieerd via `tailscale whois`). HTTP-API-eindpunten gebruiken die Tailscale-headerauthenticatie **niet**; ze volgen in plaats daarvan de normale HTTP-authenticatiemodus van de Gateway. Deze tokenloze flow gaat ervan uit dat de Gateway-host wordt vertrouwd. Is standaard `true` wanneer `tailscale.mode = "serve"`.
- `gateway.auth.rateLimit`: optionele begrenzer voor mislukte authenticatie. Wordt toegepast per client-IP en per authenticatiebereik (gedeeld geheim en apparaattoken worden onafhankelijk bijgehouden). Geblokkeerde pogingen retourneren `429` + `Retry-After`.
  - Op het asynchrone Tailscale Serve-pad van de Control UI worden mislukte pogingen voor dezelfde `{scope, clientIp}` geserialiseerd voordat de mislukking wordt weggeschreven. Gelijktijdige ongeldige pogingen van dezelfde client kunnen daardoor bij het tweede verzoek de begrenzer activeren, in plaats van dat beide als gewone afwijkingen gelijktijdig doorgaan.
  - `gateway.auth.rateLimit.exemptLoopback` is standaard `true`; stel `false` in wanneer je bewust ook snelheidsbeperking op localhostverkeer wilt toepassen (voor testconfiguraties of strikte proxy-implementaties).
- WS-authenticatiepogingen met een browseroorsprong worden altijd begrensd, waarbij de loopbackvrijstelling is uitgeschakeld (verdediging in de diepte tegen browsergebaseerde brutekrachtaanvallen op localhost).
- Op loopback worden die blokkeringen voor browseroorsprongen geïsoleerd per genormaliseerde `Origin`-
  waarde, zodat herhaalde mislukkingen vanaf één localhost-oorsprong niet automatisch
  een andere oorsprong blokkeren.
- `tailscale.mode`: `serve` (alleen tailnet, loopbackbinding) of `funnel` (openbaar, vereist authenticatie).
- `tailscale.serviceName`: optionele Tailscale-servicenaam voor de Serve-modus, zoals
  `svc:openclaw`. Wanneer deze is ingesteld, geeft OpenClaw deze door aan `tailscale serve
--service`, zodat de Control UI via een benoemde Service kan worden aangeboden in plaats
  van via de hostnaam van het apparaat. De waarde moet de indeling voor Tailscale-
  Service-namen van `svc:<dns-label>` gebruiken; bij het opstarten wordt de afgeleide Service-URL gemeld.
- `tailscale.preserveFunnel`: wanneer `true` en `tailscale.mode = "serve"`, controleert OpenClaw
  `tailscale funnel status` voordat Serve bij het opstarten opnieuw wordt toegepast en slaat
  dit over als een extern geconfigureerde Funnel-route de Gateway-poort al dekt.
  Standaard `false`.
- `controlUi.allowedOrigins`: expliciete lijst met toegestane browseroorsprongen voor Gateway-WebSocket-verbindingen. Vereist voor openbare browseroorsprongen buiten loopback. Privé-UI-ladingen met dezelfde oorsprong via LAN/Tailnet vanaf loopback-, RFC1918-/link-local-, `.local`-, `.ts.net`- of Tailscale-CGNAT-hosts worden geaccepteerd zonder terugval op Host-headers in te schakelen.
- `controlUi.toolTitles`: schakel door AI gegenereerde doeltitels voor toolaanroepen in Control UI-chat in. Standaard: `false` (toolweergave blijft volledig deterministisch zonder modelaanroepen op de achtergrond). Wanneer dit is ingeschakeld, voorziet de methode `chat.toolTitles` complexe aanroepen van labels via standaardroutering voor utiliteitsmodellen — de `utilityModel` van de agent (een beslissing van de operator die begrensde toolargumenten naar de gekozen provider kan sturen, zoals bij elke utiliteitstaak), of het gedeclareerde standaardmodel voor kleine modellen van de sessieprovider (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`) — en slaat resultaten op in de statusdatabase per agent, zodat herhaalde weergaven nooit opnieuw worden gefactureerd. `utilityModel: \"\"` schakelt titels uit zoals bij elke andere utiliteitstaak; titels vallen nooit terug op het primaire model.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: gevaarlijke modus die terugval op Host-headeroorsprong inschakelt voor implementaties die bewust vertrouwen op oorsprongsbeleid op basis van Host-headers.
- `terminal.enabled`: schakel de operator-terminal met beheerdersbereik in. Standaard: `false`. De terminal start een host-PTY in de geselecteerde agentwerkruimte, neemt de omgeving van het Gateway-proces over en wordt geweigerd voor agents met `sandbox.mode: "all"`. Schakel dit alleen in voor vertrouwde operatorimplementaties; een wijziging herstart de Gateway en werkt het contentbeveiligingsbeleid van de Control UI bij.
- `terminal.shell`: optioneel uitvoerbaar shellbestand. Wanneer dit niet is ingesteld, gebruikt OpenClaw `$SHELL` op Unix en `%ComSpec%` op Windows.
- `terminal.detachedSessionTimeoutSeconds`: hoe lang een terminalsessie blijft bestaan nadat de verbinding wegvalt (pagina opnieuw laden, laptop in slaapstand), waarbij deze opnieuw koppelbaar blijft via `terminal.attach` en de recente uitvoer opnieuw wordt afgespeeld. Standaard: `300`. Stel `0` in om sessies te beëindigen zodra hun verbinding wegvalt. Losgekoppelde sessies blijven hun opdrachten uitvoeren, dus verkort dit op gedeelde of blootgestelde hosts.
- `remote.transport`: `ssh` (standaard) of `direct` (ws/wss). Voor `direct` moet `remote.url` voor openbare hosts `wss://` zijn; niet-versleutelde `ws://` wordt alleen geaccepteerd voor loopback-, LAN-, link-local-, `.local`-, `.ts.net`- en Tailscale-CGNAT-hosts.
- `remote.remotePort`: Gateway-poort op de externe SSH-host. Is standaard `18789`; gebruik dit wanneer de lokale tunnelpoort verschilt van de externe Gateway-poort.
- `remote.tlsFingerprint`: verwachte SHA-256-certificaatvingerafdruk voor een externe `wss://`-Gateway. De macOS-app past deze toe op zowel operator-/besturingsverbindingen als verbindingen met companion-Nodes. Zonder een expliciete waarde legt macOS pas bij het eerste gebruik een pin vast nadat de normale systeemvertrouwenscontrole is geslaagd.
- `remote.sshHostKeyPolicy`: beleid voor SSH-tunnelhostsleutels van macOS. `strict` is de standaard en vereist een reeds vertrouwde sleutel. `openssh` is een expliciete inschrijving voor de effectieve OpenSSH-configuratie voor beheerde aliassen; controleer de overeenkomende SSH-instellingen van gebruiker en systeem voordat je dit gebruikt. De macOS-app en `configure-remote` zetten dit beleid bij het wijzigen van doelen terug op `strict`, tenzij er opnieuw expliciet voor wordt gekozen.
- `gateway.remote.token` / `.password` zijn referentievelden voor externe clients. Ze configureren op zichzelf geen Gateway-authenticatie.
- `gateway.push.apns.relay.baseUrl`: HTTPS-basis-URL voor het externe APNs-relais dat wordt gebruikt nadat iOS-builds met relaisondersteuning registraties naar de Gateway publiceren. Openbare App Store-builds gebruiken het gehoste OpenClaw-relais. Aangepaste relais-URL's moeten overeenkomen met een bewust afzonderlijk iOS-build-/implementatiepad waarvan de relais-URL naar dat relais verwijst.
- `gateway.push.apns.relay.timeoutMs`: time-out in milliseconden voor verzending van Gateway naar relais. Is standaard `10000`.
- Registraties met relaisondersteuning worden aan een specifieke Gateway-identiteit gedelegeerd. De gekoppelde iOS-app haalt `gateway.identity.get` op, neemt die identiteit op in de relaisregistratie en stuurt een verzendtoekenning met registratiebereik door naar de Gateway. Een andere Gateway kan die opgeslagen registratie niet hergebruiken.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: tijdelijke omgevingsoverschrijvingen voor de bovenstaande relaisconfiguratie.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: alleen voor ontwikkeling bedoelde ontsnappingsmogelijkheid voor HTTP-relais-URL's op loopback. Productierelais-URL's moeten HTTPS blijven gebruiken.
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: optionele omgevingsoverschrijving voor de ingebouwde time-out van de Gateway-WebSocket-handshake vóór authenticatie.
- `channels.<provider>.healthMonitor.enabled`: uitschakeling per kanaal voor herstarts door de statusmonitor, terwijl de globale monitor ingeschakeld blijft.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: overschrijving per account voor kanalen met meerdere accounts. Wanneer deze is ingesteld, heeft deze voorrang op de overschrijving op kanaalniveau.
- Lokale aanroeppaden van de Gateway kunnen `gateway.remote.*` alleen als terugvaloptie gebruiken wanneer `gateway.auth.*` niet is ingesteld.
- Als `gateway.auth.token` / `gateway.auth.password` expliciet via SecretRef is geconfigureerd en niet kan worden herleid, wordt de herleiding gesloten geweigerd (geen maskering door externe terugval).
- `trustedProxies`: IP-adressen van reverse proxy's die TLS beëindigen of doorgestuurde clientheaders invoegen. Vermeld alleen proxy's die je beheert. Loopbackvermeldingen blijven geldig voor proxy-/lokale-detectieconfiguraties op dezelfde host (bijvoorbeeld Tailscale Serve of een lokale reverse proxy), maar ze maken loopbackverzoeken **niet** geschikt voor `gateway.auth.mode: "trusted-proxy"`.
- `allowRealIpFallback`: wanneer `true`, accepteert de Gateway `X-Real-IP` als `X-Forwarded-For` ontbreekt. Standaard `false` voor gesloten-weigergedrag.
- `gateway.nodes.pairing.autoApproveCidrs`: optionele CIDR/IP-lijst met toegestane adressen voor het automatisch goedkeuren van een eerste koppeling van een Node-apparaat zonder aangevraagde bereiken. Deze is uitgeschakeld wanneer ze niet is ingesteld. Hiermee wordt de koppeling van operator/browser/Control UI/WebChat niet automatisch goedgekeurd en worden upgrades van rol, bereik, metadata of openbare sleutel niet automatisch goedgekeurd.
- `gateway.nodes.pairing.sshVerify`: via SSH geverifieerde automatische goedkeuring voor de eerste koppeling van een Node-apparaat (standaard: ingeschakeld). De Gateway maakt via SSH verbinding terug naar de koppelingshost (BatchMode, strikte hostsleutels) en keurt alleen goed bij een exacte overeenkomst met de apparaatsleutel `openclaw node identity`. Dezelfde minimale geschiktheid als `autoApproveCidrs`; controles zijn beperkt tot privé-/CGNAT-bronadressen, tenzij `cidrs` deze overschrijft. Stel `false` in om dit uit te schakelen, of `{ user, identity, timeoutMs, cidrs }` om het af te stemmen. Zie [Node-koppeling](/nl/gateway/pairing#ssh-verified-device-auto-approval-default).
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`: algemene allow/deny-vormgeving voor gedeclareerde node-opdrachten na evaluatie van de koppeling en de platform-allowlist. Gebruik `commands.allow` om gevaarlijke node-opdrachten zoals `camera.snap`, `camera.clip`, `screen.record`, `health.summary`, `sms.search` en `sms.send` toe te staan; `commands.deny` verwijdert een opdracht, zelfs als deze anders door een platformstandaard of expliciete toestemming zou worden opgenomen. De iOS Health-machtiging, Android SMS-machtiging en Gateway-opdrachtautorisatie staan los van elkaar. Nadat een node de lijst met gedeclareerde opdrachten heeft gewijzigd, weiger je de apparaatkoppeling en keur je deze opnieuw goed, zodat de Gateway de bijgewerkte momentopname van opdrachten opslaat.
- `gateway.tools.deny`: extra toolnamen die voor HTTP `POST /tools/invoke` worden geblokkeerd (breidt de standaardweigerlijst uit).
- `gateway.tools.allow`: verwijder toolnamen uit de standaard HTTP-weigerlijst voor
  aanroepen van eigenaren/beheerders. Dit promoveert aanroepen met een identiteit via `operator.write`
  niet tot eigenaar-/beheerderstoegang; `cron`, `gateway` en `nodes` blijven
  niet beschikbaar voor aanroepen van niet-eigenaren, zelfs wanneer ze op de allowlist staan.

</Accordion>

### OpenAI-compatibele eindpunten

- Admin HTTP RPC: standaard uitgeschakeld, net als de Plugin `admin-http-rpc`. Schakel de Plugin in om `POST /api/v1/admin/rpc` te registreren. Zie [Admin HTTP RPC](/nl/plugins/admin-http-rpc).
- Chat Completions: standaard uitgeschakeld. Schakel dit in met `gateway.http.endpoints.chatCompletions.enabled: true`.
- Responses API: `gateway.http.endpoints.responses.enabled`.
- Beveiliging van URL-invoer voor Responses:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    Lege toelatingslijsten worden als niet ingesteld beschouwd; gebruik `gateway.http.endpoints.responses.files.allowUrl=false`
    en/of `gateway.http.endpoints.responses.images.allowUrl=false` om het ophalen van URL's uit te schakelen.
- Optionele header voor extra responsbeveiliging:
  - `gateway.http.securityHeaders.strictTransportSecurity` (stel dit alleen in voor HTTPS-origins die je beheert; zie [Authenticatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth#tls-termination-and-hsts))

### Isolatie van meerdere instanties

Voer meerdere gateways op één host uit met unieke poorten en statusmappen:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

Handige vlaggen: `--dev` (gebruikt `~/.openclaw-dev` + poort `19001`), `--profile <name>` (gebruikt `~/.openclaw-<name>`).

Zie [Meerdere Gateways](/nl/gateway/multiple-gateways).

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: schakelt TLS-beëindiging bij de Gateway-listener (HTTPS/WSS) in (standaard: `false`).
- `autoGenerate`: genereert automatisch een lokaal zelfondertekend certificaat/sleutelpaar wanneer er geen expliciete bestanden zijn geconfigureerd; uitsluitend voor lokaal gebruik en ontwikkeling.
- `certPath`: bestandssysteempad naar het TLS-certificaatbestand.
- `keyPath`: bestandssysteempad naar het bestand met de privé-TLS-sleutel; beperk de toegangsrechten.
- `caPath`: optioneel pad naar een CA-bundel voor clientverificatie of aangepaste vertrouwensketens.

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: bepaalt hoe configuratiewijzigingen tijdens runtime worden toegepast.
  - `"off"`: negeer live wijzigingen; voor wijzigingen is een expliciete herstart vereist.
  - `"restart"`: start het Gateway-proces bij een configuratiewijziging altijd opnieuw.
  - `"hot"`: pas wijzigingen binnen het proces toe zonder opnieuw op te starten.
  - `"hybrid"` (standaard): probeer eerst dynamisch te herladen; val indien nodig terug op opnieuw opstarten.
- `debounceMs`: debouncevenster in ms voordat configuratiewijzigingen worden toegepast (niet-negatief geheel getal; standaard: `300`).
- `deferralTimeoutMs`: optionele maximale wachttijd in ms voor lopende bewerkingen voordat een herstart of dynamische herlading van het kanaal wordt afgedwongen. Laat dit weg om de standaard begrensde wachttijd (`300000`) te gebruiken; stel `0` in om onbeperkt te wachten en periodiek waarschuwingen te registreren dat er nog bewerkingen in behandeling zijn.

---

## Cloudworkeromgevingen

Cloudworkers zijn optioneel. Als `cloudWorkers` ontbreekt of `profiles` leeg is, accepteert OpenClaw geen nieuwe workercreaties. Eerder aangemaakte duurzame records worden nog steeds afgestemd en blijven zichtbaar; de bestaande Gateway/Node-projectie blijft ongewijzigd.

Elke workerprovider moet vanuit vertrouwde provisioningsuitvoer een SSH-`hostKey` retourneren, exact als `algorithm base64`, zonder hostnaam of opmerking. Bootstrap schrijft die sleutel naar een geïsoleerd `known_hosts`-bestand, gebruikt `StrictHostKeyChecking=yes` en stopt voordat een verbinding wordt geopend wanneer de provider de sleutel weglaat. Er is geen fallback waarbij de eerste verbinding automatisch wordt vertrouwd.

De tunnel wordt op aanvraag ingesteld en maakt geen deel uit van de provisioning. Wanneer deze wordt gestart, stuurt de Gateway een workerlokale Unix-socket via een reverse forwarding door naar het eigen loopback-WebSocket-eindpunt. De socket bevindt zich in een willekeurig toegewezen externe map die alleen toegankelijk is voor de eigenaar; in tegenstelling tot een loopback-TCP-poort is deze niet bereikbaar voor andere accounts op een worker met meerdere gebruikers en kan deze niet botsen met de poort van een andere omgeving. SSH-keepalives en een begrensde exponentiële vertraging voor nieuwe verbindingspogingen zijn alleen actief zolang de eigenaar van de tunnel actueel blijft. Bij het stoppen van de tunnel worden nieuwe verbindingspogingen geblokkeerd voordat het SSH-proces wordt gesloten.

Besturingsverkeer en werkruimteoverdracht gebruiken afzonderlijke SSH-verbindingen. Beide gebruiken dezelfde opgeloste identiteit en hetzelfde geïsoleerde, vastgezette `known_hosts`-bestand, maar de werkruimteoverdracht deelt geen multiplexing van SSH-verbindingen met de langlopende tunnel, zodat rsync het besturingsverkeer niet kan blokkeren.

### Crabbox-profiel

De meegeleverde `crabbox`-provider voorziet via de lokale Crabbox-CLI in een lease met SSH-mogelijkheden. De interne `settings.provider` selecteert de Crabbox-backend; deze staat los van de externe OpenClaw-provider-id.

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // Standaard; gebruik "npm" alleen voor een uitgebrachte Gateway-versie.
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // Optioneel absoluut pad. Standaard: naastgelegen ../crabbox/bin/crabbox, daarna PATH.
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider` (vereist): Crabbox-backend die wordt doorgegeven via `--provider`. Gebruik een backend waarvan de inspectie-uitvoer een SSH-eindpunt bevat; `aws` selecteert de directe AWS-backend.
- `settings.class` (vereist): Crabbox-machineklasse die wordt doorgegeven aan `--class`.
- `settings.ttl` en `settings.idleTimeout` (vereist): positieve Go-duurtekenreeksen die worden doorgegeven aan `--ttl` en `--idle-timeout`. Deze beveiligingsmechanismen aan de providerzijde staan los van het hieronder opgeslagen `lifetime`-beleid van OpenClaw.
- `settings.binary`: optioneel absoluut pad naar het uitvoerbare Crabbox-bestand. Zonder dit pad controleert OpenClaw eerst de naastgelegen Crabbox-checkout, daarna uitvoerbare vermeldingen in `PATH` en roept het ten slotte `crabbox` aan, zodat een ontbrekende CLI als zichtbare providerfout wordt gemeld.

Onbekende instellingen worden geweigerd. Crabbox-referenties en backendspecifieke accountconfiguratie blijven onder beheer van Crabbox; plaats ze niet in `settings`. OpenClaw roept alleen de lokale CLI aan en doet vanuit deze Plugin geen netwerkaanroepen naar de provider. Provisioning geeft altijd `--keep=true` door; OpenClaw beheert de externe levenscyclus en vernietigt de lease met `crabbox stop`.

<Note>
  OpenClaw zet het leaselokale `sshKey`-pad van Crabbox om via de providergebonden geheimresolver en zet de gezaghebbende `sshHostKey` vast die door `crabbox inspect --json` wordt geretourneerd. Voor AWS-toelating is ook `providerMetadata.instanceProfileAttached` vereist. Installeer Crabbox 0.38.1 of nieuwer voor dit gesloten inspectiecontract.
</Note>

### Statisch SSH-ontwikkelprofiel

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`: benoemde workerprofielen met niet-lege id's waarvan witruimte aan het begin en einde is verwijderd. Elk profiel selecteert een provider die door een Plugin is geregistreerd.
- `provider`: niet-lege workerprovider-id. De voorbeelden gebruiken de meegeleverde `crabbox`-provider en de QA Lab-provider `static-ssh`.
- `install`: installatiemethode voor de worker. `"bundle"` (standaard) draagt een op inhoudshash gebaseerde bundel van de geïnstalleerde Gateway-build over en ondersteunt uitgebrachte, ontwikkelings- en niet-uitgebrachte versies. `"npm"` is een optionele optimalisatie voor een ongewijzigde verpakte release; deze installeert `openclaw@<exact gateway version>` vanuit het openbare npm-register en installeert nooit `latest`.
- Meegeleverde providerplugins worden automatisch geselecteerd wanneer ze zijn geconfigureerd, maar expliciete uitschakelingen en `plugins.allow` blijven van toepassing. Neem de provider-id op (bijvoorbeeld `crabbox`) wanneer een toelatingslijst is geconfigureerd. Externe providerplugins moeten ook worden geïnstalleerd en expliciet ingeschakeld.
- `settings`: door de provider beheerde, begrensde JSON. De geselecteerde Plugin definieert en valideert de sleutels; gebruik [SecretRef-objecten](/nl/gateway/secrets) voor waarden die geheimen bevatten. De statische SSH-provider vereist `host`, `user`, `hostKey` en `keyRef`; `port` is standaard `22`. `hostKey` moet één regel met een openbare OpenSSH-hostsleutel (`algorithm base64`) zijn die van de bekende host of via een ander vertrouwd kanaal is verkregen, zonder voorvoegsel met opties.
- `lifetime.idleTimeoutMinutes`: positieve gehele minuten die worden opgeslagen voor later beleid voor terugwinning bij inactiviteit.
- `lifetime.maxLifetimeMinutes`: positieve gehele minuten die worden opgeslagen voor later levenscyclusbeleid.

Er moet al een ondersteunde Node-runtime (22.22.3+, 24.15+ of 25.9+) met SQLite die veilig is voor WAL-resets op de worker zijn geïnstalleerd. Voor de optionele methode `"npm"` zijn ook `npm` en uitgaande HTTPS-toegang tot het openbare npm-register vereist. Het instellen van toolchains met netwerktoegang valt onder het providerbeleid; bootstrap meldt een bruikbare fout in plaats van zelf toolchains te installeren.

Deze basis installeert en verifieert de Gateway-build en biedt een levenscyclus voor het starten en stoppen van de tunnel, maar start niet de algemene OpenClaw-CLI. Het zelfstandige workerstartpunt en de lus worden in de volgende cloudworkermijlpaal toegevoegd.

Elk duurzaam omgevingsrecord behoudt de gevalideerde providerinstellingen, de opgeloste installatiemethode en het levenscyclusbeleid in een momentopname van het profiel op het moment van aanmaken. Het wijzigen of verwijderen van een benoemd profiel is van invloed op nieuwe creaties; bestaande records blijven hun levenscyclus afstemmen met die momentopname, mits de verantwoordelijke Plugin beschikbaar blijft.

Levensduurwaarden zijn in de eerste cloudworkerrelease alleen gegevens; automatische handhaving wordt met later levenscycluswerk toegevoegd. Voor profielwijzigingen moet de Gateway opnieuw worden gestart.

<Warning>
  De `static-ssh`-provider is een ontwikkelharnas voor QA Lab in de bronstructuur en wordt uitgesloten van verpakte distributies. Een worker die op de gedeelde host van deze provider draait, kan niet-gerelateerde hostgegevens lezen. Gebruik deze provider daarom niet als isolatiegrens voor productie.
  De beheerder moet de verwachte `hostKey` leveren; OpenClaw leert of accepteert geen sleutel van de eerste verbinding.
  Door de lease te vernietigen, wordt alleen het logische record van OpenClaw vrijgegeven; de host wordt niet gestopt of opgeschoond.
</Warning>

---

## Hooks

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

Authenticatie: `Authorization: Bearer <token>` of `x-openclaw-token: <token>`.
Hooktokens in de querytekenreeks worden geweigerd.

Opmerkingen over validatie en veiligheid:

- `hooks.enabled=true` vereist een niet-lege `hooks.token`.
- `hooks.token` moet verschillen van actieve gedeelde-geheim-authenticatie van de Gateway (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` of `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`); bij hergebruik registreert het opstartproces een niet-fatale beveiligingswaarschuwing.
- `openclaw security audit` markeert hergebruik van hook-/Gateway-authenticatie als een kritieke bevinding, inclusief Gateway-wachtwoordauthenticatie die alleen tijdens de audit wordt opgegeven (`--auth password --password <password>`). Voer `openclaw doctor --fix` uit om een permanent opgeslagen, hergebruikte `hooks.token` te roteren en werk vervolgens externe hook-afzenders bij zodat ze het nieuwe hook-token gebruiken.
- `hooks.path` kan niet `/` zijn; gebruik een speciaal subpad zoals `/hooks`.
- Als `hooks.allowRequestSessionKey=true`, beperk dan `hooks.allowedSessionKeyPrefixes` (bijvoorbeeld `["hook:"]`).
- Als een toewijzing of voorinstelling een `sessionKey` met sjabloon gebruikt, stel dan `hooks.allowedSessionKeyPrefixes` en `hooks.allowRequestSessionKey=true` in. Voor statische toewijzingssleutels is die expliciete inschakeling niet vereist.

**Eindpunten:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - `sessionKey` uit de aanvraagpayload wordt alleen geaccepteerd wanneer `hooks.allowRequestSessionKey=true` (standaard: `false`).
- `POST /hooks/<name>` → omgezet via `hooks.mappings`
  - Door sjablonen gerenderde `sessionKey`-waarden van toewijzingen worden beschouwd als extern aangeleverd en vereisen ook `hooks.allowRequestSessionKey=true`.

<Accordion title="Toewijzingsdetails">

- `match.path` komt overeen met het subpad na `/hooks` (bijv. `/hooks/gmail` → `gmail`).
- `match.source` komt overeen met een payloadveld voor algemene paden.
- Sjablonen zoals `{{messages[0].subject}}` lezen uit de payload.
- `transform` kan verwijzen naar een JS-/TS-module die een hookactie retourneert.
  - `transform.module` moet een relatief pad zijn en blijft binnen `hooks.transformsDir` (absolute paden en padtraversal worden geweigerd).
  - Bewaar `hooks.transformsDir` onder `~/.openclaw/hooks/transforms`; Skills-mappen van de werkruimte worden geweigerd. Als `openclaw doctor` dit pad als ongeldig meldt, verplaats je de transformatiemodule naar de map voor hooktransformaties of verwijder je `hooks.transformsDir`.
- `agentId` routeert naar een specifieke agent; bij onbekende ID's wordt teruggevallen op de standaardagent.
- `allowedAgentIds`: beperkt de effectieve agentroutering, inclusief het pad naar de standaardagent wanneer `agentId` is weggelaten (`*` of weggelaten = alles toestaan, `[]` = alles weigeren).
- `defaultSessionKey`: optionele vaste sessiesleutel voor hook-agentuitvoeringen zonder expliciete `sessionKey`.
- `allowRequestSessionKey`: staat `/hooks/agent`-aanroepers en door sjablonen aangestuurde sessiesleutels van toewijzingen toe om `sessionKey` in te stellen (standaard: `false`).
- `allowedSessionKeyPrefixes`: optionele allowlist met voorvoegsels voor expliciete `sessionKey`-waarden (aanvraag + toewijzing), bijv. `["hook:"]`. Dit wordt vereist wanneer een toewijzing of voorinstelling een `sessionKey` met sjabloon gebruikt.
- `deliver: true` verzendt het definitieve antwoord naar een kanaal; `channel` is standaard `last`.
- `model` overschrijft het LLM voor deze hookuitvoering (moet toegestaan zijn als de modelcatalogus is ingesteld).

</Accordion>

### Gmail-integratie

- De ingebouwde Gmail-voorinstelling gebruikt `sessionKey: "hook:gmail:{{messages[0].id}}"`.
- Deze sleutel per bericht isoleert de gesprekscontext, niet de toegang tot tools of de werkruimte. Zonder een aangepaste toewijzing die `agentId` instelt, gebruikt de voorinstelling de standaardagent.
- Routeer Gmail voor niet-vertrouwde inboxen naar een speciale leesagent en beperk die agent met [sandbox- en toolbeleid per agent](/nl/tools/multi-agent-sandbox-tools). Als de leesagent de hoofdagent moet informeren, beperk je de overdracht met [`tools.agentToAgent`](/nl/gateway/config-tools#toolsagenttoagent). Zie [Promptinjectie](/nl/gateway/security#prompt-injection) voor het aanbevolen dreigingsmodel en modelniveau.
- Als je die routering per bericht behoudt, stel dan `hooks.allowRequestSessionKey: true` in en beperk `hooks.allowedSessionKeyPrefixes` zodat deze overeenkomt met de Gmail-naamruimte, bijvoorbeeld `["hook:", "hook:gmail:"]`.
- Als je `hooks.allowRequestSessionKey: false` nodig hebt, overschrijf je de voorinstelling met een statische `sessionKey` in plaats van de standaardsjabloon.

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- De Gateway start `gog gmail watch serve` bij het opstarten automatisch wanneer dit is geconfigureerd. Stel `OPENCLAW_SKIP_GMAIL_WATCHER=1` in om dit uit te schakelen.
- Voer geen afzonderlijke `gog gmail watch serve` uit naast de Gateway.

---

## Host voor de canvas-Plugin

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // of OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- Serveert door agents bewerkbare HTML/CSS/JS en A2UI via HTTP onder de Gateway-poort:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- Alleen lokaal: behoud `gateway.bind: "loopback"` (standaard).
- Bindingen buiten loopback: canvasroutes vereisen Gateway-authenticatie (token/wachtwoord/vertrouwde proxy), net als andere HTTP-oppervlakken van de Gateway.
- Node-WebViews verzenden doorgaans geen authenticatieheaders; nadat een Node is gekoppeld en verbonden, maakt de Gateway voor die Node bestemde capability-URL's bekend voor toegang tot canvas/A2UI.
- Capability-URL's zijn gekoppeld aan de actieve WS-sessie van de Node en verlopen snel. Er wordt geen IP-gebaseerde fallback gebruikt.
- Injecteert een client voor live herladen in geserveerde HTML.
- Maakt automatisch een eerste `index.html` aan wanneer deze leeg is.
- Serveert A2UI ook op `/__openclaw__/a2ui/`.
- Voor wijzigingen moet de Gateway opnieuw worden gestart.
- Schakel live herladen uit voor grote mappen of bij `EMFILE`-fouten.

---

## Detectie

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimaal | volledig | uit
    },
  },
}
```

- `minimal` (standaard): laat `cliPath` + `sshPort` weg uit TXT-records.
- `full`: neem `cliPath` + `sshPort` op; voor LAN-multicastadvertenties moet de meegeleverde `bonjour`-Plugin nog steeds zijn ingeschakeld.
- `off`: onderdruk LAN-multicastadvertenties zonder de inschakeling van de Plugin te wijzigen.
- De meegeleverde `bonjour`-Plugin start automatisch op macOS-hosts en moet expliciet worden ingeschakeld op Linux, Windows en gecontaineriseerde Gateway-implementaties.
- De hostnaam is standaard de systeemhostnaam wanneer die een geldig DNS-label is, met `openclaw` als fallback. Overschrijf dit met `OPENCLAW_MDNS_HOSTNAME`.
- `OPENCLAW_DISABLE_BONJOUR=1` schakelt mDNS-advertenties volledig uit en overschrijft `discovery.mdns.mode`.

### Groot gebied (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

Schrijft een unicast-DNS-SD-zone onder `~/.openclaw/dns/`. Combineer dit voor detectie tussen netwerken met een DNS-server (CoreDNS aanbevolen) + gesplitste DNS van Tailscale.

Instellen: `openclaw dns setup --apply`.

---

## Omgeving

### `env` (inline omgevingsvariabelen)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- Inline omgevingsvariabelen worden alleen toegepast als de sleutel ontbreekt in de procesomgeving.
- `.env`-bestanden: `.env` in de huidige werkmap + `~/.openclaw/.env` (geen van beide overschrijft bestaande variabelen).
- `shellEnv`: importeert ontbrekende verwachte sleutels uit het profiel van je aanmeldshell.
- Zie [Omgeving](/nl/help/environment) voor de volledige prioriteitsvolgorde.

### Vervanging van omgevingsvariabelen

Verwijs in elke configuratietekenreeks naar omgevingsvariabelen met `${VAR_NAME}`:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- Alleen namen in hoofdletters komen overeen: `[A-Z_][A-Z0-9_]*`.
- Ontbrekende/lege variabelen veroorzaken een fout bij het laden van de configuratie.
- Escape met `$${VAR}` voor een letterlijke `${VAR}`.
- Werkt met `$include`.

---

## Geheimen

Verwijzingen naar geheimen zijn additief: waarden in platte tekst blijven werken.

### `SecretRef`

Gebruik één objectvorm:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

Validatie:

- `provider`-patroon: `^[a-z][a-z0-9_-]{0,63}$`
- `source: "env"`-ID-patroon: `^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"`-ID: absolute JSON-pointer (bijvoorbeeld `"/providers/openai/apiKey"`)
- `source: "exec"`-ID-patroon: `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` (ondersteunt AWS-achtige `secret#json_key`-selectors)
- `source: "exec"`-ID's mogen geen door schuine strepen gescheiden padsegmenten `.` of `..` bevatten (bijvoorbeeld `a/../b` wordt geweigerd)

### Ondersteund referentieoppervlak

- Canonieke matrix: [Referentieoppervlak voor SecretRef-inloggegevens](/nl/reference/secretref-credential-surface)
- `secrets apply` richt zich op ondersteunde paden voor `openclaw.json`-inloggegevens.
- `auth-profiles.json`-verwijzingen worden opgenomen in runtimeomzetting en auditdekking.

### Configuratie van leveranciers van geheimen

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // optionele expliciete omgevingsprovider
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

Opmerkingen:

- `file`-provider ondersteunt `mode: "json"` en `mode: "singleValue"` (`id` moet `"value"` zijn in de singleValue-modus).
- Paden van bestands- en uitvoeringsproviders werken fail-closed wanneer verificatie van Windows-ACL's niet beschikbaar is. Stel `allowInsecurePath: true` alleen in voor vertrouwde paden die niet kunnen worden geverifieerd.
- `exec`-provider vereist een absoluut `command`-pad en gebruikt protocolpayloads op stdin/stdout.
- Paden naar opdrachten die symbolische koppelingen zijn, worden standaard geweigerd. Stel `allowSymlinkCommand: true` in om paden met symbolische koppelingen toe te staan terwijl het omgezette doelpad wordt gevalideerd.
- Als `trustedDirs` is geconfigureerd, wordt de controle op vertrouwde mappen toegepast op het omgezette doelpad.
- De onderliggende omgeving van `exec` is standaard minimaal; geef vereiste variabelen expliciet door met `passEnv`.
- Verwijzingen naar geheimen worden tijdens activering omgezet in een momentopname in het geheugen, waarna aanvraagpaden alleen de momentopname lezen.
- Tijdens activering wordt op actieve oppervlakken gefilterd: niet-omgezette verwijzingen op ingeschakelde oppervlakken laten het opstarten/herladen mislukken, terwijl inactieve oppervlakken met diagnostische meldingen worden overgeslagen.

---

## Opslag van authenticatie

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- Profielen per agent worden opgeslagen in `<agentDir>/auth-profiles.json`.
- `auth-profiles.json` ondersteunt verwijzingen op waardeniveau (`keyRef` voor `api_key`, `tokenRef` voor `token`) voor statische referentiemodi.
- Verouderde platte `auth-profiles.json`-toewijzingen zoals `{ "provider": { "apiKey": "..." } }` zijn geen runtime-indeling; `openclaw doctor --fix` herschrijft ze naar canonieke `provider:default`-API-sleutelprofielen met een `.legacy-flat.*.bak`-back-up.
- Profielen in OAuth-modus (`auth.profiles.<id>.mode = "oauth"`) ondersteunen geen door SecretRef ondersteunde referenties voor authenticatieprofielen.
- Statische runtime-referenties zijn afkomstig uit in het geheugen opgeloste snapshots; verouderde statische `auth.json`-vermeldingen worden bij detectie gewist.
- Verouderde OAuth-importen uit `~/.openclaw/credentials/oauth.json`.
- Zie [OAuth](/nl/concepts/oauth).
- Runtimegedrag voor geheimen en `audit/configure/apply`-hulpmiddelen: [Geheimenbeheer](/nl/gateway/secrets).

---

## Audit

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

De Gateway registreert auditgebeurtenissen met **alleen metadata** voor agentuitvoeringen en
toolacties in de gedeelde statusdatabase. Metadata over de levenscyclus van berichten is een
afzonderlijke optie waarvoor expliciete inschakeling vereist is. Het logboek bevat identiteit, timing, toolnamen en genormaliseerde
resultaten, maar nooit prompts, berichtinhoud, toolargumenten, resultaten of onbewerkte
fouttekst. Berichtrijen slaan geen onbewerkte platformaccount-, conversatie-,
bericht- en doel-id's op. Sessiecodes voor uitvoeringen/tools blijven beschikbaar voor correlatie
en kunnen zelf platformaccount- of peer-id's bevatten. Records
vervallen na 30 dagen en het logboek is beperkt tot 100,000 rijen. Vraag ze op met
[`openclaw audit`](/nl/cli/audit) of de
[`audit.activity.list`](/nl/gateway/protocol#audit-ledger-rpc) Gateway-RPC. Zie
[Auditgeschiedenis](/nl/gateway/audit) voor het volledige gegevensmodel, de privacysemantiek
en de beperkingen van de dekking.

- `enabled`: registreer nieuwe auditgebeurtenissen (standaard: `true`). Het logboek is
  standaard ingeschakeld, omdat een auditspoor dat pas na een incident wordt ingeschakeld
  het incident niet kan verklaren. Als je `false` instelt, worden na het opnieuw starten van de Gateway geen nieuwe gebeurtenissen meer ingevoegd;
  bestaande records blijven leesbaar totdat ze vervallen. Als je dit weer inschakelt, wordt
  de registratie vanaf dat moment hervat — het hiaat wordt niet achteraf aangevuld.
- `messages`: bereik van berichtmetadata (standaard: `"off"`). `"direct"` registreert
  alleen bekende directe gesprekken. `"all"` registreert ook groepen, kanalen en
  onbekende gesprekstypen. Beide modi blijven vrij van inhoud en vervangen onbewerkte
  identificatoren door installatiegebonden pseudoniemen met sleutel waar correlatie
  beschikbaar is. Dit zijn hulpmiddelen voor correlatie en geen anonimisering; de statusdatabase
  slaat de afleidingssleutel op, maar RPC- en CLI-exports doen dat niet.

De actieve Gateway legt `audit.enabled` en `audit.messages` vast bij het opstarten;
start deze opnieuw nadat je een van beide instellingen hebt gewijzigd. Berichtdekking omvat momenteel
geaccepteerde inkomende berichten die de kerndispatch bereiken en één terminale rij per
oorspronkelijke logische uitgaande antwoordpayload die de gedeelde duurzame bezorging bereikt.
Plugin-lokale en directe verzendpaden die deze gedeelde grenzen omzeilen, worden
nog niet gedekt. De begrensde achtergrond-
schrijver werkt volgens best effort en is geen verliesvrij compliance-archief.

---

## Logboekregistratie

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- Standaardlogbestand: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`; benoemde profielen gebruiken `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`.
- Stel `logging.file` in voor een stabiel pad.
- `consoleLevel` wordt verhoogd naar `debug` wanneer `--verbose`.
- `maxFileBytes`: maximale grootte van het actieve logbestand in bytes vóór rotatie (positief geheel getal; standaard: `104857600` = 100 MB). OpenClaw bewaart maximaal vijf genummerde archieven naast het actieve bestand.
- `redactSensitive` / `redactPatterns`: best-effortmaskering voor console-uitvoer, bestandslogboeken, OTLP-logrecords en opgeslagen tekst van sessietranscripten. `redactSensitive: "off"` schakelt alleen dit algemene beleid voor logboeken/transcripten uit; veiligheidsoppervlakken voor UI/tools/diagnostiek redigeren geheimen nog steeds vóór uitvoer.

---

## Diagnostiek

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: hoofdschakelaar voor instrumentatie-uitvoer (standaard: `true`).
- `flags`: reeks vlagtekenreeksen die gerichte loguitvoer inschakelen (ondersteunt jokertekens zoals `"telegram.*"` of `"*"`).
- `otel.enabled`: schakelt de OpenTelemetry-exportpijplijn in (standaard: `false`). Zie [OpenTelemetry-export](/nl/gateway/opentelemetry) voor de volledige configuratie, signaalcatalogus en het privacymodel.
- `otel.endpoint`: collector-URL voor OTel-export.
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`: optionele signaalspecifieke OTLP-eindpunten. Wanneer ze zijn ingesteld, overschrijven ze `otel.endpoint` alleen voor dat signaal.
- `otel.protocol`: `"http/protobuf"` (standaard) of `"grpc"`.
- `otel.headers`: aanvullende HTTP/gRPC-metadataheaders die met OTel-exportverzoeken worden verzonden.
- `otel.serviceName`: servicenaam voor resourcekenmerken.
- `otel.traces` / `otel.metrics` / `otel.logs`: schakel de export van traces, metrische gegevens of logboeken in.
- `otel.logsExporter`: bestemming voor logboekexport: `"otlp"` (standaard), `"stdout"` voor één JSON-object per stdout-regel, of `"both"`.
- `otel.sampleRate`: traceringsbemonsteringsfrequentie `0`-`1`.
- `otel.flushIntervalMs`: periodiek interval voor het wegschrijven van telemetrie in ms.
- `otel.captureContent`: expliciet in te schakelen vastlegging van onbewerkte inhoud voor OTEL-spankenmerken. Standaard uitgeschakeld. De booleaanse waarde `true` legt bericht-/toolinhoud vast die niet van het systeem afkomstig is; met de objectvorm kun je `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `systemPrompt` en `toolDefinitions` expliciet inschakelen.
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`: omgevingsschakelaar voor de nieuwste experimentele vorm van GenAI-inferentiespans, waaronder `{gen_ai.operation.name} {gen_ai.request.model}`-spannamen, het spantype `CLIENT` en `gen_ai.provider.name` in plaats van het verouderde `gen_ai.system`. Standaard behouden spans `openclaw.model.call` en `gen_ai.system` voor compatibiliteit; GenAI-metrische gegevens gebruiken begrensde semantische kenmerken.
- `OPENCLAW_OTEL_PRELOADED=1`: omgevingsschakelaar voor hosts die al een globale OpenTelemetry-SDK hebben geregistreerd. OpenClaw slaat dan het door de Plugin beheerde opstarten/afsluiten van de SDK over, terwijl diagnostische listeners actief blijven.
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` en `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`: signaalspecifieke omgevingsvariabelen voor eindpunten die worden gebruikt wanneer de overeenkomstige configuratiesleutel niet is ingesteld.
- `cacheTrace.enabled`: leg snapshots van cachetraces vast voor ingebedde uitvoeringen (standaard: `false`).
- `cacheTrace.filePath`: uitvoerpad voor cachetrace-JSONL (standaard: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`).
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: bepalen wat in de cachetrace-uitvoer wordt opgenomen (allemaal standaard: `true`).

---

## Update

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`: releasekanaal - `"stable"`, `"extended-stable"`, `"beta"` of `"dev"`. Extended-stable is uitsluitend voor pakketten: voorgrondopdrachten beheren de installatie, terwijl de Gateway alleen-lezen updatemeldingen kan geven.
- `checkOnStart`: controleer bij het starten van de Gateway op npm-updates (standaard: `true`). Opgeslagen extended-stable-selecties gebruiken dezelfde alleen-lezen melding en een meldingsinterval van 24 uur.
- `auto.enabled`: schakel automatische achtergrondupdates in voor stabiele en bèta-pakketinstallaties (standaard: `false`). Extended-stable wordt nooit automatisch toegepast.

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`: globale ACP-functieschakelaar (standaard: `true`; stel `false` in om ACP-dispatch- en spawnmogelijkheden te verbergen).
- `dispatch.enabled`: onafhankelijke schakelaar voor het dispatchen van ACP-sessiebeurten (standaard: `true`). Stel `false` in om ACP-opdrachten beschikbaar te houden en tegelijkertijd de uitvoering te blokkeren.
- `backend`: standaard-id van de ACP-runtimebackend (moet overeenkomen met een geregistreerde ACP-runtime-Plugin).
  Installeer eerst de backend-Plugin en neem, als `plugins.allow` is ingesteld, de Plugin-id van de backend op (bijvoorbeeld `acpx`); anders wordt de ACP-backend niet geladen.
- `fallbacks`: geordende lijst met id's van alternatieve ACP-backends die worden geprobeerd wanneer de primaire backend vroegtijdig faalt met een fout die tijdelijk lijkt (niet beschikbaar, snelheidsbeperkt, quotum uitgeput of overbelast), voordat deze uitvoer heeft geproduceerd. Elke vermelding moet overeenkomen met een geregistreerde backend van een ACP-runtime-Plugin.
- `defaultAgent`: id van de alternatieve ACP-doelagent wanneer bij spawns geen expliciet doel wordt opgegeven.
- `allowedAgents`: toelatingslijst met agent-id's die zijn toegestaan voor ACP-runtimesessies; leeg betekent geen aanvullende beperking.
- `stream.repeatSuppression`: onderdruk herhaalde status-/toolregels per beurt (standaard: `true`).
- `stream.deliveryMode`: `"live"` streamt incrementeel; `"final_only"` buffert tot terminale beurtgebeurtenissen.
- `stream.tagVisibility`: record van tagnamen naar booleaanse overschrijvingen van de zichtbaarheid voor gestreamde gebeurtenissen.
- `runtime.installCommand`: optionele installatieopdracht die wordt uitgevoerd bij het initialiseren van een ACP-runtimeomgeving.

---

## Wizard

Gedrag en metadata voor begeleide CLI-installatiestromen (`onboard`, `configure`, `doctor`):

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`: toestemming voor detectie die aan het begin van de begeleide onboarding wordt gekozen. Met `"full"` (aanbevolen) kan de installatie automatisch zoeken naar AI-apps, sleutels en lokale runtimes; met `"guarded"` vraagt de installatie één keer toestemming voordat er wordt gezocht en wordt in plaats daarvan handmatige configuratie aangeboden.

- `wizard.appRecommendations` is standaard ingesteld op `true`. Stel dit in op `false` om aanbevelingen voor geïnstalleerde applicaties tijdens de begeleide of klassieke onboarding uit te schakelen en Gateway-toegang tot `device.apps` te blokkeren. Node-hosts vereisen nog steeds hun afzonderlijke, standaard uitgeschakelde vlag voor het delen van geïnstalleerde apps voordat ze de opdracht bekendmaken.

---

## Identiteit

Zie de identiteitsvelden van `agents.entries` onder [Standaardinstellingen voor agents](/nl/gateway/config-agents#agent-defaults).

---

## Bridge (verouderd, verwijderd)

Huidige builds bevatten de TCP-bridge niet meer. Nodes maken verbinding via de Gateway-WebSocket. `bridge.*`-sleutels maken geen deel meer uit van het configuratieschema (validatie mislukt totdat ze zijn verwijderd; `openclaw doctor --fix` kan onbekende sleutels verwijderen).

<Accordion title="Verouderde bridgeconfiguratie (historische referentie)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // verouderde terugvaloptie voor opgeslagen taken met notify:true
    webhookToken: "replace-with-dedicated-token", // optioneel bearer-token voor uitgaande webhookauthenticatie
    sessionRetention: "24h", // duurtekenreeks of false
  },
}
```

- `sessionRetention`: hoelang voltooide geïsoleerde Cron-uitvoeringssessies worden bewaard voordat SQLite-sessierijen worden opgeschoond. Bepaalt ook de opschoning van gearchiveerde, verwijderde Cron-transcripten. Standaard: `24h`; stel in op `false` om dit uit te schakelen.
- De uitvoeringsgeschiedenis bewaart automatisch de nieuwste 2000 terminalrijen per taak. Verloren rijen behouden hun opschoningsperiode van 24 uur.
- `webhookToken`: bearer-token dat wordt gebruikt voor POST-levering via de Cron-webhook (`delivery.mode = "webhook"`); als dit wordt weggelaten, wordt geen authenticatieheader verzonden.
- `webhook`: verouderde terugval-URL voor legacy-webhooks (http/https), gebruikt door `openclaw doctor --fix` om opgeslagen taken te migreren die nog `notify: true` hebben; levering tijdens runtime gebruikt `delivery.mode="webhook"` per taak plus `delivery.to`, of `delivery.completionDestination` wanneer aankondigingslevering behouden blijft.

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: schakel foutmeldingen voor Cron-taken in (standaard: `false`).
- `after`: aantal opeenvolgende fouten voordat een melding wordt geactiveerd (positief geheel getal, min.: `1`).
- `cooldownMs`: minimaal aantal milliseconden tussen herhaalde meldingen voor dezelfde taak (niet-negatief geheel getal).
- `includeSkipped`: tel opeenvolgende overgeslagen uitvoeringen mee voor de meldingsdrempel (standaard: `false`). Overgeslagen uitvoeringen worden afzonderlijk bijgehouden en hebben geen invloed op de back-off bij uitvoeringsfouten.
- `mode`: leveringsmodus - `"announce"` verzendt via een kanaalbericht; `"webhook"` plaatst een bericht op de geconfigureerde webhook.
- `accountId`: optionele account- of kanaal-id om de levering van meldingen te beperken.

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- Standaardbestemming voor Cron-foutmeldingen voor alle taken.
- `mode`: `"announce"` of `"webhook"`; wordt standaard ingesteld op `"announce"` wanneer voldoende doelgegevens beschikbaar zijn.
- `channel`: kanaaloverschrijving voor aankondigingslevering. `"last"` hergebruikt het laatst bekende leveringskanaal.
- `to`: expliciet aankondigingsdoel of webhook-URL. Vereist voor de webhookmodus.
- `accountId`: optionele accountoverschrijving voor levering.
- `delivery.failureDestination` per taak overschrijft deze globale standaardinstelling.
- Wanneer er geen globale of taakspecifieke foutbestemming is ingesteld, vallen taken die al via `announce` leveren bij een fout terug op dat primaire aankondigingsdoel.
- `delivery.failureDestination` wordt alleen ondersteund voor `sessionTarget="isolated"`-taken, tenzij de primaire `delivery.mode` van de taak `"webhook"` is.

Zie [Cron-taken](/nl/automation/cron-jobs). Geïsoleerde Cron-uitvoeringen worden bijgehouden als [achtergrondtaken](/nl/automation/tasks).

## Sjabloonvariabelen voor mediamodellen

Sjabloonplaatsaanduidingen die worden uitgebreid in `tools.media.models[].args`:

| Variabele                    | Beschrijving                                       |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | Volledige hoofdtekst van het inkomende bericht                         |
| `{{RawBody}}`               | Onbewerkte hoofdtekst (zonder geschiedenis-/afzenderwrappers)             |
| `{{BodyStripped}}`          | Hoofdtekst waaruit groepsvermeldingen zijn verwijderd                 |
| `{{From}}`                  | Afzender-id                                 |
| `{{To}}`                    | Bestemmings-id                            |
| `{{MessageSid}}`            | Kanaalbericht-id                                |
| `{{SessionId}}`             | UUID van de huidige sessie                              |
| `{{IsNewSession}}`          | `"true"` wanneer een nieuwe sessie is aangemaakt                 |
| `{{AttachmentUrl}}`         | URL van de huidige bijlage of providerreferentie      |
| `{{AttachmentPath}}`        | Lokaal pad van de huidige bijlage                     |
| `{{AttachmentContentType}}` | MIME-inhoudstype van de huidige bijlage              |
| `{{AttachmentDir}}`         | Map die `AttachmentPath` bevat             |
| `{{AttachmentIndex}}`       | Op nul gebaseerde index van het bronfeit                      |
| `{{Transcript}}`            | Audiotranscript                                  |
| `{{Prompt}}`                | Omgezette mediaprompt voor CLI-vermeldingen             |
| `{{MaxChars}}`              | Omgezet maximumaantal uitvoertekens voor CLI-vermeldingen         |
| `{{ChatType}}`              | `"direct"` of `"group"`                           |
| `{{GroupSubject}}`          | Groepsonderwerp (naar beste vermogen)                       |
| `{{GroupMembers}}`          | Voorbeeldweergave van groepsleden (naar beste vermogen)               |
| `{{SenderName}}`            | Weergavenaam van afzender (naar beste vermogen)                 |
| `{{SenderE164}}`            | Telefoonnummer van afzender (naar beste vermogen)                 |
| `{{Provider}}`              | Providerhint (whatsapp, telegram, discord enz.) |

De verouderde namen `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` en `{{MediaDir}}`
blijven beschikbaar tijdens de compatibiliteitsperiode van de Plugin-SDK, maar zijn
verouderd. Nieuwe configuraties moeten de variabelen van `Attachment*` gebruiken.

---

## Configuratie-insluitingen (`$include`)

Splits de configuratie op in meerdere bestanden:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**Samenvoeggedrag:**

- Eén bestand: vervangt het bevattende object.
- Bestandsarray: wordt in volgorde diep samengevoegd (latere waarden overschrijven eerdere).
- Sleutels op hetzelfde niveau: worden na insluitingen samengevoegd (en overschrijven ingesloten waarden).
- Geneste insluitingen: maximaal 10 niveaus diep.
- Paden: worden relatief ten opzichte van het insluitende bestand omgezet, maar moeten binnen de configuratiemap op het hoogste niveau blijven (`dirname` van `openclaw.json`). Absolute/`../`-vormen zijn alleen toegestaan wanneer ze nog steeds binnen die grens worden omgezet. Stel `OPENCLAW_INCLUDE_ROOTS` (absolute paden) in om aanvullende hoofdmappen buiten de configuratiemap toe te staan.
- Limieten: paden mogen geen null-bytes bevatten en moeten vóór en na omzetting strikt korter zijn dan 4096 tekens; elk ingesloten bestand is beperkt tot 2 MB.
- Door OpenClaw beheerde schrijfbewerkingen die slechts één sectie op het hoogste niveau wijzigen die door een insluiting van één bestand wordt ondersteund, schrijven door naar dat ingesloten bestand. `plugins install` werkt bijvoorbeeld `plugins: { $include: "./plugins.json5" }` bij in `plugins.json5` en laat `openclaw.json` intact.
- Hoofdinsluitingen, insluitingsarrays en insluitingen met overschrijvingen op hetzelfde niveau zijn alleen-lezen voor door OpenClaw beheerde schrijfbewerkingen; deze schrijfbewerkingen mislukken gesloten in plaats van de configuratie af te vlakken.
- Fouten: duidelijke berichten voor ontbrekende bestanden, parseerfouten, circulaire insluitingen, ongeldige padindelingen en buitensporige lengte.

---

## Gerelateerd

- [Configuratie](/nl/gateway/configuration)
- [Configuratievoorbeelden](/nl/gateway/configuration-examples)
- [Doctor](/nl/gateway/doctor)
