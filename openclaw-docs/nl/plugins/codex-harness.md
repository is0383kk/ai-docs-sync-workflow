---
read_when:
    - Je wilt de officiële Codex-app-serverharnas gebruiken
    - Je hebt voorbeelden van Codex-harnasconfiguraties nodig
    - Je wilt dat implementaties met alleen Codex mislukken in plaats van terug te vallen op OpenClaw
summary: Voer beurten van de ingebouwde OpenClaw-agent uit via de officiële Codex-app-serverharnas
title: Codex-harnas
x-i18n:
    generated_at: "2026-07-27T05:12:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e016a1689af65c5520d529ce22a87bd25ee29369f7aedca77b27f943a7f21b0f
    source_path: plugins/codex-harness.md
    workflow: 16
---

De officiële `codex`-plugin voert ingebedde OpenAI-agentbeurten uit via de Codex
app-server in plaats van via de ingebouwde OpenClaw-harness. Codex beheert de
agentensessie op laag niveau: native hervatting van threads, native voortzetting van tools,
native Compaction en uitvoering via de app-server. OpenClaw beheert nog steeds chatkanalen,
sessiebestanden, modelselectie, dynamische OpenClaw-tools, goedkeuringen,
medialevering en de zichtbare transcriptspiegel.

Gebruik canonieke OpenAI-modelreferenties zoals `openai/gpt-5.6-sol`. Configureer geen
verouderde Codex GPT-referenties; plaats de authenticatievolgorde voor OpenAI-agents onder `auth.order.openai`.
Verouderde profiel-id's voor Codex-authenticatie en verouderde vermeldingen voor de Codex-authenticatievolgorde worden
hersteld door `openclaw doctor --fix`.

Als het runtimebeleid voor provider/model niet is ingesteld of `auto` is, selecteert alleen het voorvoegsel `openai/*`
deze harness nooit. OpenAI mag Codex alleen impliciet selecteren voor een
exacte officiële HTTPS-route voor Platform Responses of ChatGPT Responses zonder
handmatig ingestelde aanvraagoverschrijving. Zie
[Impliciete OpenAI-agentruntime](/nl/providers/openai#implicit-agent-runtime).
Als Codex de authenticatie beheert voordat bekend is of de routering via Platform of ChatGPT verloopt, vereist OpenClaw
nog steeds dat elke kandidaatroute compatibiliteit met Codex declareert. Alleen
native beheer van authenticatie omzeilt die routecontrole nooit.

Als er geen OpenClaw-sandbox actief is, start OpenClaw Codex-app-serverthreads
met de native codemodus van Codex ingeschakeld (alleen-codemodus blijft standaard uitgeschakeld), zodat
native werkruimte- en codemogelijkheden beschikbaar blijven naast dynamische
OpenClaw-tools die via de app-serverbrug `item/tool/call` worden gerouteerd. Een
actieve OpenClaw-sandbox of beperkt toolbeleid schakelt de native codemodus
volledig uit, tenzij je je aanmeldt voor het experimentele exec-serverpad van de sandbox.

Met de standaardwaarde `tools.exec.host: "auto"` en zonder actieve OpenClaw-sandbox
ontvangt Codex ook de tools `node_exec` en `node_process` voor opdrachten op gekoppelde
Nodes. De native shell blijft op de host en in de werkruimte van de Codex-app-server
(lokaal bij de Gateway voor de standaard stdio-implementatie); `node_exec` selecteert een Node op
naam of id en handhaaft het goedkeuringsbeleid voor Nodes van OpenClaw. Als een eindige
runtime-toelatingslijst de native codemodus uitschakelt en de beurt zonder
uitvoeringsomgeving achterlaat, houdt OpenClaw in plaats daarvan de door beleid gefilterde tools `exec` en `process`
beschikbaar voor directe uitvoering zonder sandbox.

Deze native Codex-functie staat los van
[OpenClaw-codemodus](/nl/tools/code-mode), een optionele QuickJS-WASI-runtime
voor algemene OpenClaw-uitvoeringen met een andere invoervorm voor `exec`. Begin voor de
bredere scheiding tussen model, provider en runtime bij
[Agentruntimes](/nl/concepts/agent-runtimes): `openai/gpt-5.6-sol` is de modelreferentie,
`codex` is de runtime en Telegram, Discord, Slack of een ander
kanaal is het communicatieoppervlak.

## Vereisten

- De officiële `@openclaw/codex`-plugin is geïnstalleerd. Neem `codex` op in
  `plugins.allow` als je configuratie een toelatingslijst gebruikt.
- Een stabiele Codex-app-server van `0.143.0` tot en met `0.145.0`. De plugin beheert standaard een compatibel
  binair bestand, zodat een opdracht `codex` op `PATH` het normale
  opstarten niet beïnvloedt.
- Codex-authenticatie via `openclaw models auth login --provider openai`, een
  app-serveraccount dat al aanwezig is in de Codex-home van de agent, of een
  expliciet Codex-authenticatieprofiel met API-sleutel.

Zie voor de authenticatieprioriteit, omgevingsisolatie, aangepaste app-serveropdrachten,
modeldetectie en de volledige lijst met configuratievelden
[Referentie voor de Codex-harness](/nl/plugins/codex-harness-reference).

## Snel starten

Installeer de officiële plugin en meld je vervolgens aan met Codex OAuth:

```bash
openclaw plugins install @openclaw/codex
openclaw models auth login --provider openai
```

Schakel de `codex`-plugin in en selecteer een OpenAI-agentmodel:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Als je configuratie `plugins.allow` gebruikt, voeg daar dan ook `codex` toe:

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Start de Gateway opnieuw nadat je de pluginconfiguratie hebt gewijzigd. Als een chat al een
sessie heeft, voer dan eerst `/new` of `/reset` uit, zodat de volgende beurt de harness
op basis van de huidige configuratie bepaalt.

## Threads delen met Codex Desktop en CLI

De standaardwaarde `appServer.homeScope: "agent"` isoleert elke OpenClaw-agent van
de native Codex-status van de beheerder. Als je een eigenaar dezelfde
native threads wilt laten inspecteren en beheren die in Codex Desktop en de Codex CLI worden weergegeven, schakel je het
gebruik van de Codex-home van de gebruiker in:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            homeScope: "user",
          },
        },
      },
    },
  },
}
```

De gebruikershomemodus ondersteunt een lokaal beheerd stdio-proces of het gedeelde Unix-sockettransport.
Deze gebruikt `$CODEX_HOME` wanneer dat is ingesteld en anders `~/.codex`, inclusief
de native Codex-authenticatie, configuratie, plugins en threadopslag van die home. OpenClaw
injecteert geen OpenClaw-authenticatieprofiel in deze app-server.

Beurten van de eigenaar krijgen de tool `codex_threads`: native threads weergeven, doorzoeken,
lezen, forken, hernoemen, archiveren en herstellen. Fork een thread om deze voort te zetten in
OpenClaw; de fork wordt gekoppeld aan de huidige OpenClaw-sessie en blijft
zichtbaar voor andere native Codex-clients. Voor archivering is expliciete
bevestiging vereist dat de thread elders is gesloten. Als supervisie ook is
ingeschakeld, vereisen transcriptvelden en mutaties de bijbehorende
aanmelding via `supervision.allowRawTranscripts` of `supervision.allowWriteControls`.

Hervat of beschrijf dezelfde thread niet gelijktijdig via onafhankelijke beheerde
stdio-app-servers. Codex coördineert actieve schrijvers binnen één app-server, niet
tussen afzonderlijke processen. Forken is het veilige co-existentiepad voor gewone
stdio-sessies in de gebruikershome.

Alleen `appServer.homeScope: "user"` beheert de vlootcatalogus niet. Native
sessiedetectie is ingeschakeld zolang de plugin actief is; stel
`sessionCatalog.enabled: false` in om deze uit de OpenClaw-zijbalk te verwijderen zonder
Codex uit te schakelen. De catalogus gebruikt een afzonderlijke supervisieverbinding; zonder
expliciete verbindingsinstellingen voor `appServer` gebruikt die verbinding standaard beheerde
stdio in de gebruikershome, terwijl de gewone harness agentspecifiek blijft. Expliciete
instellingen voor `appServer` worden door beide paden gerespecteerd. Stel `homeScope: "user"`
expliciet in, zoals hierboven, wanneer de gewone harness ook native status moet delen.

## Toezicht houden op Codex-sessies

Dezelfde `codex`-plugin kan niet-gearchiveerde Codex-sessies van de Gateway-
computer en aangemelde gekoppelde Nodes weergeven. Een opgeslagen of inactieve sessie die lokaal bij de Gateway wordt uitgevoerd, kan
een modelvergrendelde chat maken die de begrensde, opgeslagen geschiedenis van de gebruiker en assistent
spiegelt. De privébinding gebruikt de supervisieverbinding voor de native
momentopname, canonieke vertakking en latere beurten, terwijl gewone Codex-sessies
agentspecifiek blijven. De eerste canonieke start gebruikt exact het model en de provider die
Codex voor de fork van de momentopname retourneert. Bij latere hervattingen wordt de selectie aan de native
configuratie van Codex overgelaten; het buitenste OpenClaw-model en de fallbackketen vervangen
deze nooit. Opgeslagen en inactieve rijen kunnen worden gearchiveerd na expliciete bevestiging
dat er geen andere uitvoerder actief is. Actieve bronnen kunnen geen vertakking maken en niet worden gearchiveerd; een bestaande
chat onder supervisie kan nog steeds worden geopend. Sessies op gekoppelde Nodes blijven beperkt tot metadata.

Zie [Toezicht houden op Codex-sessies](/plugins/codex-supervision) voor installatie, vertakkingsregels,
beperkingen voor gekoppelde Nodes, blootstelling van metadata en probleemoplossing.

## Configuratie

| Behoefte                                            | Instellen                                                                                         | Waar                               |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------- |
| De harness inschakelen                              | `plugins.entries.codex.enabled: true`                                                                                | OpenClaw-configuratie              |
| Detectie van native Codex-sessies verbergen         | `plugins.entries.codex.config.sessionCatalog.enabled: false`                                                                                | Configuratie van de Codex-plugin   |
| Een plugininstallatie met toelatingslijst behouden  | Neem `codex` op in `plugins.allow`                                                  | OpenClaw-configuratie              |
| Geschikte OpenAI-beurten impliciet Codex laten gebruiken | Exacte officiële HTTPS-route voor Responses/ChatGPT, geen handmatig ingestelde aanvraagoverschrijving, runtime niet ingesteld/`auto` | OpenAI-provider-/modelconfiguratie |
| Aanmelden met ChatGPT/Codex OAuth                   | `openclaw models auth login --provider openai`                                                                                | CLI-authenticatieprofiel           |
| Reserve-API-sleutel voor Codex-uitvoeringen toevoegen | `openai:*`-profiel met API-sleutel, vermeld na abonnementsauthenticatie in `auth.order.openai` | CLI-authenticatieprofiel + OpenClaw-configuratie |
| Gesloten falen wanneer Codex niet beschikbaar is    | Provider of model `agentRuntime.id: "codex"`                                                              | OpenClaw-model-/providerconfiguratie |
| Rechtstreeks OpenAI API-verkeer gebruiken           | Provider of model `agentRuntime.id: "openclaw"` met normale OpenAI-authenticatie                             | OpenClaw-model-/providerconfiguratie |
| App-servergedrag afstemmen                          | `plugins.entries.codex.config.appServer.*`                                                                                | Configuratie van de Codex-plugin   |
| Native Codex-pluginapps inschakelen                 | `plugins.entries.codex.config.codexPlugins.*`                                                                                | Configuratie van de Codex-plugin   |
| Codex Computer Use inschakelen                      | `plugins.entries.codex.config.computerUse.*`                                                                                | Configuratie van de Codex-plugin   |

Geef de voorkeur aan `auth.order.openai` voor een volgorde met eerst het abonnement en de API-sleutel als reserve.
Bestaande verouderde profiel-id's voor Codex-authenticatie en de verouderde Codex-authenticatievolgorde zijn
verouderde status die alleen voor doctor is bestemd; schrijf geen nieuwe verouderde Codex GPT-referenties.

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Voor een effectieve, met Codex compatibele route blijven beide bovenstaande profielen kandidaten
voor dezelfde Codex-uitvoering. De profielvolgorde kiest de referenties, niet de runtime.
Het wijzigen van de authenticatievolgorde maakt een aangepaste route, Completions-route, HTTP-route of
route met aanvraagoverschrijving niet compatibel met Codex.

### Compaction

Stel `compaction.model` of `compaction.provider` niet in voor agents die door Codex worden ondersteund.
Codex voert Compaction uit via de native threadstatus van de app-server, zodat
OpenClaw die lokale overschrijvingen van de samenvatter tijdens runtime negeert en
`openclaw doctor --fix` deze verwijdert wanneer de agent Codex gebruikt.

Lossless blijft ondersteund als contextengine voor samenstelling, opname en
onderhoud rond Codex-beurten, geconfigureerd via
`plugins.slots.contextEngine: "lossless-claw"` en
`plugins.entries.lossless-claw.config.summaryModel`, niet via
`agents.defaults.compaction.provider`. `openclaw doctor --fix` migreert de
oude vorm `compaction.provider: "lossless-claw"` naar de sleuf voor de Lossless-
contextengine wanneer Codex de actieve runtime is, maar native Codex blijft
Compaction beheren. De native app-serverharness ondersteunt contextengines
die samenstelling vóór de prompt nodig hebben; algemene CLI-backends, waaronder `codex-cli`,
bieden die hostmogelijkheid niet.

Voor agents die door Codex worden ondersteund, start `/compact` native Compaction via de Codex-app-server
op de gekoppelde thread en wacht op het eindresultaat. Het gedeelde
budget `agents.defaults.compaction.timeoutSeconds` is van toepassing; bij een time-out
vraagt OpenClaw Codex de native beurt te onderbreken en houdt het de afscherming per thread
in stand totdat de beëindiging is bevestigd. Er wordt nooit teruggevallen op een contextengine of
openbare OpenAI-samenvatter. Als de native Codex-threadbinding ontbreekt of
verouderd is, faalt de opdracht gesloten in plaats van stilzwijgend van Compaction-
backend te wisselen.

### Lange context via directe API

Codex-abonnement en direct OpenAI API-verkeer zijn afzonderlijke contracten. De
live ChatGPT/Codex-catalogus biedt doorgaans een modelvenster van `272000` tokens,
terwijl OpenAI voor GPT-5.5 en GPT-5.6 een Platform API-venster van `1050000` tokens
en een maximale uitvoer van `128000` documenteert. Als de volledige uitvoerruimte
wordt gereserveerd, blijft een afgeleid invoerbudget van `922000` tokens over.
Voor aanvragen met meer dan `272000` invoertokens geldt OpenAI's hogere prijsstelling
voor lange context.

Begin met een volledige Codex-modelcatalogus die compatibel is met de geïnstalleerde
Codex-versie. Behoud voor elke directe GPT-5.5- of GPT-5.6-vermelding die lange context
moet gebruiken de rest van de descriptor en stel het volgende in:

```json
{
  "context_window": 922000,
  "max_context_window": 922000,
  "auto_compact_token_limit": 700000
}
```

Codex past zijn normale reserve van 95% voor het effectieve venster toe op de
cataloguswaarde `922000` en rapporteert daarom ongeveer `875900`
bruikbare tokens. Compaction bij `700000` laat `175900` tokens over
vóór die effectieve beveiligingsgrens en `222000` vóór de invoerruimte die
veilig is voor de provider. Deze grotere marge is bewust gekozen: Codex controleert
de reeds vastgelegde context voordat het volgende gebruikersbericht en contextupdates
worden toegevoegd. De drempel moet daarom zowel één grote inkomende beurt als tools,
instructies, serialisatie en de Compaction-beurt zelf kunnen opvangen.

Voor zelfstandig gebruik van de Codex CLI of Desktop kan een aangepaste provider
met opdrachtauthenticatie de API-sleutel uit een systeem-sleutelhanger of geheimenbeheerder
lezen, terwijl de normale ChatGPT-aanmelding beschikbaar blijft voor connectors:

```toml
model = "gpt-5.6-terra"
model_provider = "openai_api_direct"
model_context_window = 922000
model_auto_compact_token_limit = 700000
model_auto_compact_token_limit_scope = "total"
model_catalog_json = "/absolute/path/to/models-api-1m.json"

[model_providers.openai_api_direct]
name = "OpenAI API direct"
base_url = "https://api.openai.com/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.openai_api_direct.auth]
command = "/absolute/path/to/read-openai-inference-key"
timeout_ms = 5000
refresh_interval_ms = 300000
```

De authenticatiehelper mag alleen de sleutel naar stdout schrijven. Plaats deze niet in TOML.

Behoud voor de OpenClaw Codex app-server-harnastest de standaard agentspecifieke Codex-home
en laat OpenClaw een API-sleutelprofiel voor `openai` injecteren. Geef de catalogus
en contextlimieten door als native argumenten van de Codex app-server:

```json5
{
  auth: {
    order: {
      openai: ["openai:api-key"],
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            args: [
              "app-server",
              "--listen",
              "stdio://",
              "-c",
              'model_catalog_json="/absolute/path/to/models-api-1m.json"',
              "-c",
              "model_context_window=922000",
              "-c",
              "model_auto_compact_token_limit=700000",
              "-c",
              "model_auto_compact_token_limit_scope=total",
            ],
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-terra",
      models: {
        "openai/gpt-5.6-terra": { agentRuntime: { id: "codex" } },
      },
    },
  },
}
```

Vervang `openai:api-key` indien nodig door de daadwerkelijke profiel-id van de API-sleutel. De
app-server met agentbereik ontvangt alleen die voorbereide sleutel; de systeemeigen
`~/.codex` ChatGPT-aanmelding, plugins, connectors en threadopslag van de operator blijven
ongewijzigd. Codex app-server `0.144.6` koppelt bij app-serverbeurten geen bearer-token
van een aangepaste provider voor commandoauthenticatie, dus gebruik voor deze route het hierboven
geïnjecteerde API-sleutelpad in plaats van `homeScope: "user"`.

Start na het wijzigen van de catalogus- of app-serverargumenten de Gateway opnieuw en
begin een nieuwe chat. Bestaande systeemeigen threads behouden hun vastgelegde provider-
en modelinstellingen. Controleer de runtime met `/status` en `/codex status` en
verstuur vervolgens een onschadelijke directe API-beurt voordat je een lange sessie start.

<Warning>
Een lange context is bewust opt-in. OpenAI brengt voor de volledige aanvraag 2×
het invoertarief en 1,5× het uitvoertarief in rekening zodra de invoer meer dan `272000` tokens bevat. De API blijft
doorslaggevend voor toegang, daadwerkelijke limieten en facturering. Zie
[OpenAI-modellimieten](https://developers.openai.com/api/docs/models/compare) en
[API-prijzen](https://developers.openai.com/api/docs/pricing).
</Warning>

De rest van deze pagina behandelt de implementatievorm, fail-closed-routering, het
goedkeuringsbeleid van de guardian, systeemeigen Codex-plugins en Computer Use. Raadpleeg voor volledige lijsten
met opties, standaardwaarden, enums, detectie, omgevingsisolatie, time-outs en
app-servertransportvelden de
[Codex-harnasreferentie](/nl/plugins/codex-harness-reference).

## Codex-runtime controleren

Gebruik `/status` in de chat waarin je Codex verwacht. Een OpenAI-
agentbeurt met Codex als backend toont:

```text
Runtime: OpenAI Codex
```

Controleer vervolgens de status van Codex app-server:

```text
/codex status
/codex models
/codex binding
```

`/codex binding` rapporteert de gekoppelde native thread en de huidige modelinstellingen.
`/codex status` rapporteert de connectiviteit met de app-server, het account, de frequentielimieten, MCP-
servers en skills. `/codex models` vermeldt de live Codex-app-servercatalogus
voor de harness en het account. Als `/status` onverwacht is, raadpleeg je
[Probleemoplossing](#troubleshooting).

## Routering en modelselectie

Houd providerreferenties en runtimebeleid gescheiden:

- Gebruik `openai/gpt-*` voor de canonieke selectie van OpenAI-modellen. Alleen het voorvoegsel
  selecteert nooit Codex.
- Als runtime niet is ingesteld of `auto` is, kan alleen een exacte officiële HTTPS-route voor Platform Responses
  of ChatGPT Responses zonder handmatig ingestelde aanvraagoverschrijving Codex
  impliciet selecteren.
- Gebruik geen verouderde Codex GPT-referenties in de configuratie; voer `openclaw doctor --fix` uit om
  verouderde referenties en achterhaalde vastgezette sessieroutes te herstellen.
- `agentRuntime.id: "codex"` maakt Codex een fail-closed vereiste voor een
  compatibele route. Het maakt een incompatibele effectieve route niet compatibel.
- `agentRuntime.id: "openclaw"` laat een provider of model gebruikmaken van de ingebedde
  OpenClaw-runtime wanneer dat de bedoeling is.
- `/codex ...` beheert native Codex-app-servergesprekken vanuit de chat.
- ACP/acpx is een afzonderlijk extern harnesspad. Gebruik dit alleen wanneer de gebruiker
  om ACP/acpx of een externe harnessadapter vraagt.

| Bedoeling van de gebruiker                                  | Gebruik                                                                                                |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| De huidige chat koppelen                                   | `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`                    |
| Een bestaande Codex-thread hervatten                       | `/codex resume <thread-id>`                                                                           |
| Codex-threads weergeven of filteren                         | `/codex threads [filter]`                                                                             |
| Het native doel van de gekoppelde thread lezen of bijwerken | `/codex goal [status\|set <objective>\|pause\|resume\|block\|complete\|clear]`                        |
| Native Codex-plugins weergeven                              | `/codex plugins list`                                                                                 |
| Een geconfigureerde native Codex-plugin in- of uitschakelen | `/codex plugins enable <name>`, `/codex plugins disable <name>`                                       |
| Een opgeslagen Codex CLI-sessie hervatten als een gekoppelde-nodebeurt | `/codex sessions --host <node> [filter]`, daarna `/codex resume <session-id> --host <node> --bind here` |
| Niet-gearchiveerde Codex-sessies op verschillende computers bekijken | Schakel Codex-toezicht in en open **Codex-sessies**                                                  |
| Het model, de snelle modus of de machtigingen van de gekoppelde thread wijzigen | `/codex model <model>`, `/codex fast [on\|off\|status]`, `/codex permissions [default\|yolo\|status]` |
| De actieve beurt stoppen of bijsturen                       | `/codex stop`, `/codex steer <text>`                                                                  |
| De huidige koppeling losmaken                               | `/codex detach` (alias `/codex unbind`)                                                               |
| Alleen Codex-feedback verzenden                             | `/codex diagnostics [note]`                                                                           |
| Een ACP/acpx-taak starten                                   | ACP/acpx-sessieopdrachten, niet `/codex`                                                               |

| Gebruikssituatie                                | Configureren                                                                                                | Verifiëren                              | Opmerkingen                                |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------ |
| Geschikte OpenAI-route met native Codex-runtime | Exacte officiële HTTPS-route voor Responses/ChatGPT zonder handmatig ingestelde aanvraagoverschrijving, plus ingeschakelde `codex`-plugin | `/status` toont `Runtime: OpenAI Codex` | Impliciet pad wanneer runtime niet is ingesteld/`auto` |
| Fail-closed als Codex niet beschikbaar is        | Provider of model `agentRuntime.id: "codex"`                                                                | De beurt mislukt in plaats van ingebedde fallback | Gebruik dit voor implementaties die uitsluitend Codex gebruiken |
| Rechtstreeks OpenAI-API-sleutelverkeer via OpenClaw | Provider of model `agentRuntime.id: "openclaw"` en normale OpenAI-authenticatie                                      | `/status` toont de OpenClaw-runtime        | Gebruik dit alleen wanneer OpenClaw bewust wordt ingezet |
| Verouderde configuratie                         | verouderde Codex GPT-referenties                                                                            | `openclaw doctor --fix` herschrijft deze     | Schrijf nieuwe configuratie niet op deze manier |
| ACP/acpx-Codex-adapter                          | ACP `sessions_spawn({ runtime: "acp" })`                                                                    | ACP-taak-/sessiestatus                  | Staat los van de native Codex-harness      |

`agents.defaults.imageModel` volgt dezelfde opsplitsing op basis van het voorvoegsel. Gebruik `openai/gpt-*`
voor de normale OpenAI-route en `codex/gpt-*` alleen wanneer beeldherkenning
via een begrensde Codex-app-serverbeurt moet worden uitgevoerd. Doctor herschrijft verouderde
Codex GPT-referenties naar `openai/gpt-*`.

## Implementatiepatronen

### Basisimplementatie van Codex

Gebruik de snelstartconfiguratie voor een OpenAI-model waarvan de effectieve officiële HTTPS-
route geschikt is om Codex impliciet te selecteren:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

### Implementatie met meerdere providers

Behoud Claude als de standaardagent en voeg een benoemde Codex-agent toe:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

De `main`-agent gebruikt het normale providerpad. De `codex`-agent gebruikt de Codex-
app-server wanneer de effectieve OpenAI-route compatibel blijft; voeg expliciet
modelgebonden `agentRuntime.id: "codex"` toe wanneer dit een fail-closed
vereiste moet zijn.

### Fail-closed Codex-implementatie

Een geschikte, exact overeenkomende officiële HTTPS-route van OpenAI kan naar Codex worden omgezet wanneer de
meegeleverde plugin beschikbaar is. Voeg expliciet runtimebeleid toe voor een vastgelegde
fail-closed regel:

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: {
          id: "codex",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Wanneer Codex wordt afgedwongen, stopt OpenClaw vroegtijdig als de effectieve route niet als
Codex-compatibel is gedeclareerd, de plugin is uitgeschakeld, de app-server te oud is of de
app-server niet kan worden gestart.

## App-serverbeleid

Standaard start de plugin het door OpenClaw beheerde Codex-binaire bestand lokaal met
stdio-transport. Stel `appServer.command` alleen in om bewust een
ander uitvoerbaar bestand te gebruiken. Codex classificeert WebSocket-transport als experimenteel
en niet-ondersteund; gebruik het alleen voor niet-productietests met een app-server
die al elders draait:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
          },
        },
      },
    },
  },
}
```

Lokale stdio-app-serversessies gebruiken standaard de vertrouwde houding voor de lokale operator:
`approvalPolicy: "never"`, `approvalsReviewer: "user"` en
`sandbox: "danger-full-access"`. Als lokale Codex-vereisten die
impliciete YOLO-houding niet toestaan, selecteert OpenClaw in plaats daarvan toegestane
Guardian-machtigingen. Wanneer voor de sessie een OpenClaw-sandbox actief is, schakelt OpenClaw
voor die beurt de native Code Mode van Codex, MCP-servers van de gebruiker en de uitvoering van
door apps ondersteunde plugins uit, in plaats van te vertrouwen op sandboxing aan de hostzijde van Codex.
Shelltoegang verloopt dan via dynamische tools die door de OpenClaw-sandbox worden ondersteund,
zoals `sandbox_exec` en `sandbox_process`, wanneer de normale exec-/procestools
beschikbaar zijn.

Gebruik de genormaliseerde exec-modus van OpenClaw voor native automatische review door Codex,
vóór sandboxontsnappingen of extra machtigingen:

```json5
{
  tools: {
    exec: {
      mode: "auto",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Voor Codex-app-serversessies wordt `tools.exec.mode: "auto"` toegewezen aan
door Codex Guardian beoordeelde goedkeuringen: doorgaans `approvalPolicy: "on-request"`,
`approvalsReviewer: "auto_review"` en `sandbox: "workspace-write"` wanneer
lokale vereisten die waarden toestaan. In `tools.exec.mode: "auto"`
behoudt OpenClaw verouderde onveilige Codex-overschrijvingen `approvalPolicy: "never"` of
`sandbox: "danger-full-access"` niet; gebruik `tools.exec.mode: "full"` voor
een bewuste Codex-houding zonder goedkeuringen. De verouderde voorinstelling
`plugins.entries.codex.config.appServer.mode: "guardian"` werkt nog steeds,
maar `tools.exec.mode: "auto"` is het genormaliseerde OpenClaw-oppervlak.

Zie [Machtigingsmodi](/nl/tools/permission-modes) voor de vergelijking op modusniveau met
exec-goedkeuringen op de host en ACPX-machtigingen. Zie
[Codex-harnasreferentie](/nl/plugins/codex-harness-reference) voor elk
app-serverveld, de authenticatievolgorde, omgevingsisolatie en het time-outgedrag.

## Opdrachten en diagnostiek

De plugin `codex` registreert `/codex` als slashopdracht op elk kanaal dat
OpenClaw-tekstopdrachten ondersteunt.

Voor native uitvoering en beheer is een eigenaar of een `operator.admin`-Gateway-client
vereist: threads koppelen of hervatten, beurten verzenden of stoppen,
het model, de snelle modus of de machtigingsstatus wijzigen, compacten of beoordelen en
een koppeling losmaken. Andere geautoriseerde afzenders behouden alleen-lezen opdrachten voor
status, hulp, account, model, thread, native doel, MCP-server, skill en inspectie van koppelingen.

Veelgebruikte vormen:

- `/codex status` controleert de app-serververbinding, modellen, het account, gebruikslimieten,
  MCP-servers en Skills.
- `/codex models` vermeldt actieve Codex-app-servermodellen.
- `/codex threads [filter]` vermeldt recente Codex-app-serverthreads.
- `/codex goal` leest of wijzigt het native Codex-doel van de gekoppelde thread. Automatische voortzetting van doelen door Codex blijft uitgeschakeld; OpenClaw beheert nog geen autonome vervolgbeurten.
- `/codex resume <thread-id>` koppelt de huidige OpenClaw-sessie aan een
  bestaande Codex-thread.
- `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`
  koppelt de huidige chat.
- `/codex detach` (of `/codex unbind`) maakt de huidige koppeling los.
- `/codex binding` beschrijft de huidige koppeling.
- `/codex stop` stopt de actieve beurt; `/codex steer <text>` stuurt deze bij.
- `/codex model <model>`, `/codex fast [on|off|status]` en
  `/codex permissions [default|yolo|status]` wijzigen de status per gesprek.
- `/codex compact` vraagt de Codex-app-server om de gekoppelde thread te compacten.
- `/codex review` start een native Codex-review voor de gekoppelde thread.
- `/codex diagnostics [note]` vraagt om bevestiging voordat Codex-feedback voor de
  gekoppelde thread wordt verzonden.
- `/codex account` toont de accountstatus en gebruikslimieten.
- `/codex mcp` vermeldt de status van MCP-servers van de Codex-app-server.
- `/codex skills` vermeldt Skills van de Codex-app-server.
- `/codex plugins list`, `/codex plugins enable <name>` en
  `/codex plugins disable <name>` beheren geconfigureerde native Codex-plugins.
- `/codex computer-use [status|install]` beheert Codex Computer Use.
- `/codex help` toont de volledige opdrachtstructuur.

Begin voor de meeste ondersteuningsmeldingen met `/diagnostics [note]` in het
gesprek waarin de fout optrad. Hiermee wordt één Gateway-diagnoserapport gemaakt en wordt voor
Codex-harnassessies om goedkeuring gevraagd om de relevante Codex-feedbackbundel te
verzenden. Zie [Diagnostiek exporteren](/nl/gateway/diagnostics) voor het privacymodel en
het gedrag in groepschats. Gebruik `/codex diagnostics [note]` alleen wanneer je specifiek
de Codex-feedback voor de momenteel gekoppelde thread wilt uploaden zonder
de volledige Gateway-diagnostiekbundel.

### Codex-threads lokaal inspecteren

De snelste manier om een mislukte Codex-uitvoering te inspecteren, is vaak om de native
Codex-thread rechtstreeks te openen:

```bash
codex resume <thread-id>
```

Haal de thread-id op uit het voltooide antwoord van `/diagnostics`, `/codex binding`
of `/codex threads [filter]`.

Zie [Codex-harnasruntime](/nl/plugins/codex-harness-runtime#codex-feedback-upload) voor
uploadmechanismen en diagnostiekgrenzen op runtimeniveau.

### Authenticatievolgorde

In de standaardhomedirectory per agent wordt authenticatie in deze volgorde geselecteerd:

1. Geordende OpenAI-authenticatieprofielen voor de agent, bij voorkeur onder
   `auth.order.openai`. Voer `openclaw doctor --fix` uit om oudere verouderde
   Codex-authenticatieprofiel-id's en de verouderde Codex-authenticatievolgorde te migreren.
2. Het bestaande app-serveraccount in de Codex-homedirectory van die agent.
3. Alleen voor lokale stdio-starts van de app-server: `CODEX_API_KEY`, daarna
   `OPENAI_API_KEY`, wanneer er geen app-serveraccount aanwezig is en OpenAI-authenticatie
   nog steeds vereist is.

Wanneer OpenClaw een Codex-authenticatieprofiel in de stijl van een ChatGPT-abonnement aantreft,
verwijdert het `CODEX_API_KEY` en `OPENAI_API_KEY` uit het gestarte
Codex-kindproces. Zo blijven API-sleutels op Gateway-niveau beschikbaar voor embeddings of
rechtstreekse OpenAI-modellen, zonder dat native Codex-app-serverbeurten per ongeluk
via de API worden gefactureerd. Expliciete Codex-profielen met API-sleutels en de lokale
stdio-terugval op omgevingssleutels gebruiken app-serveraanmelding in plaats van overgenomen
omgevingsvariabelen van het kindproces. WebSocket-verbindingen met de app-server ontvangen geen
terugval op Gateway-API-sleutels uit de omgeving; gebruik een expliciet authenticatieprofiel of het
eigen account van de externe app-server.

Als een abonnementsprofiel een Codex-gebruikslimiet bereikt, registreert OpenClaw de
resettijd wanneer Codex die meldt en probeert het voor dezelfde Codex-uitvoering het volgende
geordende authenticatieprofiel. Nadat de resettijd is verstreken, komt het abonnementsprofiel
weer in aanmerking zonder het geselecteerde `openai/gpt-*`-model of de Codex-runtime
te wijzigen.

Wanneer native Codex-plugins zijn geconfigureerd, installeert of vernieuwt OpenClaw
die plugins via de verbonden app-server voordat apps van plugins aan de Codex-thread
beschikbaar worden gesteld. `app/list` blijft de bron van waarheid voor app-id's,
toegankelijkheid en metadata, maar OpenClaw beheert de inschakelbeslissing per thread:
als het beleid een vermelde toegankelijke app toestaat, verzendt OpenClaw
`thread/start.config.apps[appId].enabled = true`, zelfs wanneer `app/list`
momenteel meldt dat die app is uitgeschakeld. Dit pad verzint geen app-installatie
voor onbekende id's; OpenClaw activeert alleen marketplace-plugins met
`plugin/install` en vernieuwt daarna de inventaris.

### Omgevingsisolatie

Voor lokale stdio-starts van de app-server stelt OpenClaw `CODEX_HOME` in op een
directory per agent, zodat Codex-configuratie, authenticatie-/accountbestanden, plugin-cache/-gegevens
en native threadstatus standaard niet de persoonlijke
`~/.codex` van de operator lezen of beschrijven. OpenClaw behoudt de normale
procesvariabele `HOME`; subprocessen die door Codex worden uitgevoerd, kunnen
nog steeds configuratie en tokens in de homedirectory van de gebruiker vinden, en
Codex kan gedeelde vermeldingen in `$HOME/.agents/skills` en
`$HOME/.agents/plugins/marketplace.json` ontdekken. Met
`appServer.homeScope: "user"` gebruikt OpenClaw in plaats daarvan de native Codex-homedirectory
van de gebruiker en het bestaande account, zonder een OpenClaw-authenticatieprofiel te injecteren.

Als een implementatie aanvullende omgevingsisolatie vereist, voeg je die
variabelen toe aan `appServer.clearEnv`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` beïnvloedt alleen het gestarte kindproces van de Codex-app-server.
OpenClaw verwijdert `CODEX_HOME` en `HOME` tijdens lokale
startnormalisatie uit deze lijst: `CODEX_HOME` blijft naar het geselecteerde
agent- of gebruikersbereik verwijzen en `HOME` blijft overgenomen, zodat subprocessen
de normale status uit de homedirectory van de gebruiker kunnen gebruiken.

### Dynamische tools en zoeken op het web

Dynamische Codex-tools gebruiken standaard het laden via `searchable`. OpenClaw stelt normaal
geen dynamische tools beschikbaar die native werkruimtebewerkingen van Codex dupliceren:
`read`, `write`, `edit`, `apply_patch`, `exec`, `process`, `update_plan`,
`get_goal`, `create_goal`, `update_goal`, `tool_call`, `tool_describe`,
`tool_search` en `tool_search_code`. Doelbewerkingen blijven native voor Codex,
dus OpenClaw projecteert geen tweede doelopslag in Codex-beurten. De meeste
overige OpenClaw-integratietools, zoals berichten, media, Cron,
browser, Nodes, Gateway en `heartbeat_respond`, zijn beschikbaar via
het zoeken naar Codex-tools onder de naamruimte `openclaw`, waardoor de initiële
modelcontext kleiner blijft. De shellterugval voor beperkte beurten is de uitzondering voor
`exec` en `process` wanneer een eindige toelatingslijst de native Code Mode uitschakelt;
runtime-toelatingslijsten en `codexDynamicToolsExclude` blijven van toepassing.

Tools die zijn gemarkeerd als `catalogMode: "direct-only"`, waaronder de OpenClaw-tool
`computer`, gebruiken in plaats daarvan de naamruimte `openclaw_direct`.
Codex behandelt die naamruimte als `DirectModelOnly`, zodat die tools in normale threads
en threads met alleen Code Mode rechtstreeks zichtbaar blijven voor het model, in plaats van
geneste Code Mode-aanroepen naar `tools.*` te doorkruisen.

Zoeken op het web gebruikt standaard de gehoste tool `web_search` van Codex wanneer zoeken is
ingeschakeld en geen beheerde provider is geselecteerd. Native gehost zoeken en
de beheerde dynamische tool `web_search` van OpenClaw sluiten elkaar uit, zodat
beheerd zoeken de native domeinbeperkingen niet kan omzeilen. OpenClaw gebruikt de
beheerde tool wanneer gehost zoeken niet beschikbaar is, expliciet is uitgeschakeld of
is vervangen door een geselecteerde beheerde provider. OpenClaw houdt de zelfstandige
Codex-extensie `web.run` uitgeschakeld, omdat productie-app-serververkeer
de door de gebruiker gedefinieerde naamruimte `web` weigert. `tools.web.search.enabled: false`
schakelt beide paden uit, evenals LLM-only-uitvoeringen waarbij tools zijn uitgeschakeld. Codex behandelt
`"cached"` als een voorkeur en zet deze voor onbeperkte app-serverbeurten om in
actieve externe toegang. Automatische beheerde terugval faalt gesloten wanneer native
`allowedDomains` zijn ingesteld, zodat de toelatingslijst niet kan worden omzeild.
Permanente wijzigingen van het effectieve zoekbeleid roteren de gekoppelde Codex-thread
vóór de volgende beurt; tijdelijke beperkingen per beurt gebruiken een tijdelijke
beperkte thread en behouden de bestaande koppeling om deze later te hervatten.

`sessions_yield`, `sessions_spawn` en bronantwoorden die uitsluitend voor de berichtentool bestemd zijn, blijven
direct omdat het contracten voor beurtbesturing of delegatie zijn. De richtlijnen
geven nog steeds de voorkeur aan de ingebouwde `spawn_agent` van Codex als het primaire oppervlak voor Codex-subagents,
terwijl expliciete delegatie via OpenClaw of ACP rechtstreeks aanroepbaar blijft via
`sessions_spawn`. In Codex Code Mode zijn generieke dynamische-toolresultaten van OpenClaw
JSON-tekst in plaats van JavaScript-objecten; parse daarom resultaten die op JSON lijken
voordat je velden uitleest. Codex serialiseert ook geneste
dynamische aanroepen; dien meerdere `sessions_spawn`-aanroepen in een begrensde lus in
in plaats van te verwachten dat `Promise.all` ze gelijktijdig start. Reeds geaccepteerde
subagents kunnen nog steeds overlappen terwijl latere aanroepen worden ingediend. Zie
[Swarm](/tools/swarm#use-swarm-from-other-harnesses) voor een volledig patroon.
Samenwerkingsinstructies voor Heartbeat
dragen Codex op om naar `heartbeat_respond` te zoeken voordat een Heartbeat-beurt wordt beëindigd
wanneer de tool nog niet is geladen.

Stel `codexDynamicToolsLoading: "direct"` alleen in wanneer je verbinding maakt met een aangepaste
Codex-app-server die niet naar uitgestelde dynamische tools kan zoeken, of bij het
debuggen van de volledige toolpayload.

### Configuratievelden

Ondersteunde Codex-pluginvelden op het hoogste niveau:

| Veld                       | Standaard      | Betekenis                                                                                |
| -------------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| `codexDynamicToolsLoading` | `"searchable"` | Gebruik `"direct"` om dynamische OpenClaw-tools rechtstreeks in de initiële Codex-toolcontext te plaatsen. |
| `codexDynamicToolsExclude` | `[]`           | Aanvullende namen van dynamische OpenClaw-tools die uit Codex-app-serverbeurten moeten worden weggelaten. |
| `codexPlugins`             | uitgeschakeld  | Ingebouwde ondersteuning voor Codex-plugins/apps voor gemigreerde, vanuit de bron geïnstalleerde, beheerde plugins. |
| `sessionCatalog`           | ingeschakeld   | Detectie in de zijbalk voor ingebouwde Codex-sessies op deze Gateway en geschikte gekoppelde nodes. |
| `supervision`              | uitgeschakeld  | Beleid voor transcript- en schrijfrechten van ingebouwde sessies voor agents.            |

Ondersteunde `appServer`-velden:

| Veld                                          | Standaardwaarde                                        | Betekenis                                                                                                                                                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` start Codex; expliciete `"unix"` maakt verbinding met de lokale besturingssocket; `"websocket"` maakt verbinding met `url`.                                                                                                                                                                                                                                 |
| `homeScope`                                   | `"agent"`                                              | `"agent"` isoleert de normale harnasstatus per OpenClaw-agent. `"user"` is een expliciete opt-in die de native `$CODEX_HOME` of `~/.codex` deelt, native authenticatie gebruikt en threadbeheer alleen voor de eigenaar inschakelt. Het gebruikersbereik ondersteunt lokale stdio- of Unix-transport. Voor de afzonderlijke supervisieverbinding wordt een niet-ingestelde waarde omgezet in `"user"` voor stdio of Unix en `"agent"` voor WebSocket. |
| `command`                                     | beheerd Codex-binair bestand                            | Uitvoerbaar bestand voor stdio-transport. Laat dit niet ingesteld om het beheerde binaire bestand te gebruiken; stel het alleen in voor een expliciete overschrijving.                                                                                                                                                                                                                           |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | Argumenten voor stdio-transport.                                                                                                                                                                                                                                                                                                                                                                |
| `url`                                         | niet ingesteld                                         | WebSocket-URL van de app-server of `unix://`-URL. Een expliciet leeg Unix-pad selecteert de canonieke besturingssocket in de thuismap van de gebruiker.                                                                                                                                                                                                                                 |
| `authToken`                                   | niet ingesteld                                         | Bearer-token voor WebSocket-transport. Accepteert een letterlijke tekenreeks of SecretInput, zoals `${CODEX_APP_SERVER_TOKEN}`.                                                                                                                                                                                                                                                                          |
| `headers`                                     | `{}`                                                   | Extra WebSocket-headers. Headerwaarden accepteren letterlijke tekenreeksen of SecretInput-waarden, bijvoorbeeld `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`.                                                                                                                                                                                                                                                             |
| `clearEnv`                                    | `[]`                                                   | Namen van extra omgevingsvariabelen die uit het gestarte stdio-app-serverproces worden verwijderd nadat OpenClaw de overgenomen omgeving heeft opgebouwd. OpenClaw behoudt de geselecteerde `CODEX_HOME` en de overgenomen `HOME` voor lokale starts.                                                                                                                          |
| `codeModeOnly`                                | `false`                                                | Schakel het uitsluitend op codemodus gerichte tooloppervlak van Codex in. Normale dynamische OpenClaw-tools blijven beschikbaar via geneste `tools.*`-aanroepen; `openclaw_direct`-tools blijven rechtstreeks zichtbaar voor het model.                                                                                                                                                  |
| `remoteWorkspaceRoot`                         | niet ingesteld                                         | Hoofdmap van de werkruimte van de externe Codex-app-server. Wanneer deze is ingesteld, leidt OpenClaw de lokale hoofdmap van de werkruimte af uit de omgezette OpenClaw-werkruimte, behoudt het huidige cwd-achtervoegsel onder deze externe hoofdmap en stuurt alleen de uiteindelijke cwd van de app-server naar Codex. Als de cwd buiten de omgezette hoofdmap van de OpenClaw-werkruimte ligt, weigert OpenClaw veilig in plaats van een gateway-lokaal pad naar de externe app-server te sturen. |
| `requestTimeoutMs`                            | `60000`                                                | Time-out voor aanroepen van het besturingsvlak van de app-server.                                                                                                                                                                                                                                                                                                                              |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | Stiltevenster nadat Codex een beurt accepteert of na een app-serververzoek binnen een beurt, terwijl OpenClaw op `turn/completed` wacht.                                                                                                                                                                                                                                                       |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | Stiltevenster nadat een definitief/niet-commentaaritem van de assistent of een onbewerkte voltooiing van de assistent vóór een tool de vrijgave van assistentuitvoer activeert, terwijl OpenClaw nog op `turn/completed` wacht. Door dit te verhogen krijgt Codex meer tijd om `turn/completed` uit te voeren voordat OpenClaw onderbreekt en de sessiebaan vrijgeeft.                              |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | Bewaking van inactiviteit na voltooiing en voortgang die wordt gebruikt na een tooloverdracht, voltooiing van een native tool, onbewerkte assistentvoortgang na een tool, voltooiing van onbewerkte redenering of voortgang van redenering terwijl OpenClaw op `turn/completed` wacht. Gebruik dit voor vertrouwde of zware werklasten waarbij synthese na een tool terecht langer stil kan blijven dan het vrijgavebudget voor de definitieve assistent. |
| `mode`                                        | `"yolo"` tenzij lokale Codex-vereisten YOLO niet toestaan | Voorinstelling voor uitvoering met YOLO of beoordeling door een bewaker. Lokale stdio-vereisten waarin `danger-full-access`, `never`-goedkeuring of de `user`-beoordelaar ontbreekt, maken de impliciete standaardinstelling tot bewaker.                                                                                                                                       |
| `approvalPolicy`                              | `"never"` of een toegestaan goedkeuringsbeleid van de bewaker | Native Codex-goedkeuringsbeleid dat naar het starten/hervatten van threads en beurten wordt gestuurd. Standaardinstellingen van de bewaker geven de voorkeur aan `"on-request"` wanneer dit is toegestaan.                                                                                                                                                                                     |
| `sandbox`                                     | `"danger-full-access"` of een toegestane sandbox van de bewaker | Native Codex-sandboxmodus die naar het starten/hervatten van threads wordt gestuurd. Standaardinstellingen van de bewaker geven de voorkeur aan `"workspace-write"` wanneer dit is toegestaan, anders aan `"read-only"`. Wanneer een OpenClaw-sandbox actief is, gebruiken `danger-full-access`-beurten Codex `workspace-write` met netwerktoegang die is afgeleid van de uitgaande-verbindingsinstelling van de OpenClaw-sandbox. |
| `approvalsReviewer`                           | `"user"` of een toegestane beoordelaar van de bewaker | Gebruik `"auto_review"` om Codex native goedkeuringsprompts te laten beoordelen wanneer dit is toegestaan, anders `guardian_subagent` of `user`. `guardian_subagent` blijft een verouderde alias.                                                                                                                                                                                       |
| `serviceTier`                                 | niet ingesteld                                         | Optionele servicelaag van de Codex-app-server. `"priority"` schakelt routering in snelle modus in, `"flex"` vraagt flexibele verwerking aan, `null` wist de overschrijving en de verouderde `"fast"` wordt geaccepteerd als `"priority"`.                                                                                                               |
| `networkProxy`                                | uitgeschakeld                                          | Schakel netwerken via het Codex-machtigingsprofiel in voor app-serveropdrachten. OpenClaw definieert de geselecteerde `permissions.<profile>.network`-configuratie en selecteert deze met `default_permissions` in plaats van `sandbox` te verzenden.                                                                                                                                                       |
| `experimental.sandboxExecServer`              | `false`                                                | Preview-opt-in die een door een OpenClaw-sandbox ondersteunde Codex-omgeving registreert bij de ondersteunde Codex-app-server, zodat native Codex-uitvoering binnen de actieve OpenClaw-sandbox kan plaatsvinden.                                                                                                                                                                                 |

`appServer.networkProxy` is expliciet omdat dit het sandboxcontract van Codex
wijzigt. Wanneer dit is ingeschakeld, stelt OpenClaw ook `features.network_proxy.enabled`
en `default_permissions` in de Codex-threadconfiguratie in, zodat het gegenereerde
machtigingsprofiel door Codex beheerd netwerkgebruik kan starten. Standaard genereert OpenClaw
een botsingsbestendige `openclaw-network-<fingerprint>`-profielnaam
op basis van de profielinhoud; gebruik `profileName` alleen wanneer een stabiele lokale naam
vereist is.

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              unixSockets: {
                "/tmp/proxy.sock": "allow",
                "/tmp/blocked.sock": "none",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
}
```

Als de normale app-serverruntime `danger-full-access` zou zijn, gebruikt het inschakelen
van `networkProxy` bestandssysteemtoegang in werkruimtestijl voor het gegenereerde
machtigingsprofiel: door Codex beheerde netwerkhandhaving is gesandboxte
netwerktoegang, dus een profiel met volledige toegang zou uitgaand verkeer niet beschermen.
Domeinvermeldingen gebruiken `allow` of `deny`; Unix-socketvermeldingen gebruiken de
waarden `allow` of `none` van Codex.

### Dynamische time-outs voor toolaanroepen

Dynamische toolaanroepen die eigendom zijn van OpenClaw worden onafhankelijk begrensd van
`appServer.requestTimeoutMs`: Codex-`item/tool/call`-verzoeken gebruiken standaard een
OpenClaw-waakhond van 90 seconden. Een positief `timeoutMs`-argument per aanroep
verlengt of verkort het specifieke toolbudget, met een maximum van 600000 ms.
De tool `image_generate` gebruikt `agents.defaults.mediaModels.image.timeoutMs`
wanneer de toolaanroep geen eigen time-out opgeeft, en anders een standaardwaarde van 120 seconden
voor het genereren van afbeeldingen. De tool `image` voor mediabegrip
gebruikt de `timeoutSeconds` van de geselecteerde afbeeldingsgeschikte `tools.media.models[]`-vermelding of de standaardwaarde van 60 seconden voor media; voor
afbeeldingsbegrip geldt die time-out voor het verzoek zelf en wordt deze niet
verminderd door eerder voorbereidend werk. Bij een time-out breekt OpenClaw het toolsignaal
waar mogelijk af en retourneert het een mislukt dynamisch-toolantwoord aan Codex,
zodat de beurt kan doorgaan in plaats van de sessie in `processing` achter te laten.
Deze waakhond vormt het buitenste dynamische `item/tool/call`-budget; providerspecifieke
verzoektime-outs worden binnen die aanroep uitgevoerd en behouden hun eigen time-outsemantiek.

Nadat Codex een beurt accepteert en nadat OpenClaw reageert op een tot de beurt beperkt
app-serververzoek, verwacht het harnas dat Codex voortgang boekt in de huidige beurt
en de systeemeigen beurt uiteindelijk voltooit met `turn/completed`. Als de
app-server gedurende `appServer.turnCompletionIdleTimeoutMs` stil blijft, probeert OpenClaw
de Codex-beurt zo goed mogelijk te onderbreken, registreert het een diagnostische time-out en
geeft het de OpenClaw-sessiebaan vrij, zodat volgende chatberichten niet
achter een verouderde systeemeigen beurt in de wachtrij blijven staan. De meeste niet-terminale meldingen voor
dezelfde beurt schakelen die korte waakhond uit, omdat Codex heeft aangetoond dat de beurt
nog actief is.

Tooloverdrachten gebruiken een langer inactiviteitsbudget na de tool: nadat OpenClaw een
`item/tool/call`-antwoord retourneert, nadat systeemeigen toolitems zoals
`commandExecution` zijn voltooid, na onbewerkte `custom_tool_call_output`-voltooiingen
en na onbewerkte assistentvoortgang na de tool, onbewerkte redeneervoltooiingen
of redeneervoortgang. De bewaking gebruikt
`appServer.postToolRawAssistantCompletionIdleTimeoutMs` wanneer dit is geconfigureerd en
standaard anders vijf minuten; hetzelfde budget verlengt ook de
voortgangswaakhond voor het stille synthesetijdvenster voordat Codex de
volgende gebeurtenis van de huidige beurt uitstuurt. Algemene app-servermeldingen, zoals
updates van frequentielimieten, stellen de voortgang bij inactiviteit van de beurt niet opnieuw in. Redeneervoltooiingen,
voltooiingen van commentary-`agentMessage` en onbewerkte redenering vóór de tool of
assistentvoortgang kunnen worden gevolgd door een automatisch definitief antwoord, dus gebruiken ze
de antwoordbewaking na voortgang in plaats van de sessiebaan
onmiddellijk vrij te geven.

Alleen voltooide definitieve/niet-commentary `agentMessage`-items en onbewerkte
assistentvoltooiingen vóór de tool activeren de vrijgave na assistentuitvoer: als Codex daarna
stil blijft zonder `turn/completed`, probeert OpenClaw de systeemeigen
beurt zo goed mogelijk te onderbreken en geeft het de sessiebaan vrij. Als een andere beurtbewaking die vrijgavewedstrijd
wint, accepteert OpenClaw het voltooide definitieve assistentitem alsnog zodra er geen
systeemeigen verzoek, item of dynamische toolvoltooiing meer actief is en de
vrijgave na assistentuitvoer nog steeds bij het laatst voltooide item hoort, zonder
latere itemvoltooiing. Zo kan het definitieve antwoord na
voltooid toolwerk behouden blijven zonder de beurt opnieuw af te spelen. Gedeeltelijke assistentdelta's,
verouderde eerdere antwoorden en lege latere voltooiingen komen niet in aanmerking.

Veilig opnieuw afspeelbare stdio-app-serverfouten, waaronder time-outs door inactiviteit bij het voltooien van een beurt
zonder bewijs van assistentactiviteit, toolactiviteit, actieve items of neveneffecten, worden
eenmaal opnieuw geprobeerd met een nieuwe app-serverpoging. Onveilige time-outs stellen de
vastgelopen app-serverclient alsnog buiten gebruik en geven de OpenClaw-sessiebaan vrij; ze
wissen ook de verouderde systeemeigen threadkoppeling in plaats van deze
automatisch opnieuw af te spelen. Time-outs van de voltooiingsbewaking tonen Codex-specifieke
time-outtekst: veilig opnieuw afspeelbare gevallen vermelden dat het antwoord mogelijk onvolledig is, terwijl onveilige
gevallen de gebruiker vragen de huidige status te controleren voordat die het opnieuw probeert. Openbare
time-outdiagnostiek bevat structurele velden zoals de methode van de laatste
app-servermelding, de id/het type/de rol van het onbewerkte assistentresponsitem, aantallen actieve
verzoeken/items en de status van geactiveerde bewaking; wanneer de laatste melding een
onbewerkt assistentresponsitem is, bevat deze ook een begrensd tekstvoorbeeld
van de assistent. De diagnostiek bevat geen onbewerkte prompt- of toolinhoud.

### Omgevingsoverschrijvingen voor lokaal testen

- `OPENCLAW_CODEX_APP_SERVER_BIN` omzeilt het beheerde binaire bestand wanneer
  `appServer.command` niet is ingesteld.
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` is verwijderd. Gebruik in plaats daarvan
`plugins.entries.codex.config.appServer.mode: "guardian"`, of
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` voor een eenmalige lokale test. Configuratie
heeft de voorkeur voor herhaalbare implementaties, omdat het gedrag van de plugin
dan in hetzelfde beoordeelde bestand staat als de rest van de Codex-harnasconfiguratie.

## Systeemeigen Codex-plugins

Ondersteuning voor systeemeigen Codex-plugins gebruikt de eigen app- en pluginmogelijkheden
van de Codex-app-server in dezelfde Codex-thread als de OpenClaw-harnasbeurt. OpenClaw
vertaalt Codex-plugins niet naar synthetische dynamische OpenClaw-tools
van het type `codex_plugin_*`.

`codexPlugins` is alleen van invloed op sessies die het systeemeigen Codex-harnas selecteren.
Het heeft geen effect op ingebouwde harnasuitvoeringen, normale uitvoeringen van de OpenAI-provider, ACP-
gesprekskoppelingen of andere harnassen.

Minimale gemigreerde configuratie:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

De app-configuratie van de thread wordt berekend wanneer OpenClaw een Codex-harnassessie
opzet of een verouderde Codex-threadkoppeling vervangt; deze wordt niet bij
elke beurt opnieuw berekend. Gebruik na het wijzigen van `codexPlugins` `/new`, `/reset`, of start
de Gateway opnieuw, zodat toekomstige Codex-harnassessies met de bijgewerkte appset
beginnen.

Zie voor migratiegeschiktheid, app-inventaris, beleid voor destructieve acties,
verzoeken om aanvullende invoer en diagnostiek van systeemeigen plugins
[Systeemeigen Codex-plugins](/nl/plugins/codex-native-plugins).

Toegang tot apps en plugins aan de zijde van OpenAI wordt beheerd door het aangemelde Codex-
account en, voor Business- en Enterprise/Edu-werkruimten, door de app-
instellingen van de werkruimte. Zie
[Codex gebruiken met je ChatGPT-abonnement](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)
voor het overzicht van OpenAI over accounts en werkruimte-instellingen.

## Computergebruik

Computergebruik heeft een eigen installatiehandleiding:
[Codex-computergebruik](/nl/plugins/codex-computer-use).

Kort samengevat: OpenClaw levert de app voor desktopbesturing niet mee en voert
desktopacties niet zelf uit. Het bereidt de Codex-app-server voor, controleert of de
MCP-server `computer-use` beschikbaar is en laat Codex vervolgens tijdens beurten
in Codex-modus de systeemeigen MCP-toolaanroepen beheren.

## Runtimegrenzen

Het Codex-harnas wijzigt alleen de ingebedde agentexecutor op laag niveau.

- Dynamische OpenClaw-tools worden ondersteund. Codex vraagt OpenClaw om
  die tools uit te voeren, zodat OpenClaw deel blijft uitmaken van het uitvoeringspad.
- Systeemeigen shell-, patch-, MCP- en app-tools van Codex worden door Codex beheerd.
  OpenClaw kan geselecteerde systeemeigen gebeurtenissen via de
  ondersteunde relay waarnemen of blokkeren, maar herschrijft geen systeemeigen toolargumenten.
- Codex beheert systeemeigen Compaction. OpenClaw houdt een transcriptspiegel bij voor
  kanaalgeschiedenis, zoeken, `/new`, `/reset` en toekomstige wisselingen van model of harnas,
  maar vervangt Codex Compaction niet door een samenvatter van OpenClaw of
  de contextengine.
- Mediageneratie, mediabegrip, TTS, goedkeuringen en uitvoer van berichtentools
  blijven verlopen via de overeenkomende provider-/modelinstellingen van OpenClaw.
- `tool_result_persist` is van toepassing op transcripttoolresultaten die eigendom zijn van OpenClaw,
  niet op systeemeigen toolresultaatrecords van Codex.

Zie voor hooklagen, ondersteunde V1-oppervlakken, afhandeling van systeemeigen machtigingen, besturing
van wachtrijen, het uploadmechanisme voor Codex-feedback en details over Compaction
[Runtime van het Codex-harnas](/nl/plugins/codex-harness-runtime).

## Problemen oplossen

**Codex verschijnt niet als een normale `/model`-provider:** dit wordt verwacht bij nieuwe
configuraties. Selecteer een `openai/gpt-*`-model, schakel
`plugins.entries.codex.enabled` in en controleer of `plugins.allow`
`codex` uitsluit.

**OpenClaw gebruikt het ingebouwde harnas in plaats van Codex:** controleer of de effectieve
route exact een officiële HTTPS Platform Responses- of ChatGPT Responses-route is,
geen zelf opgegeven verzoekoverschrijving bevat en of de Codex-plugin is geïnstalleerd en
ingeschakeld. Alleen het voorvoegsel `openai/gpt-*` is niet voldoende. Stel voor sluitend bewijs tijdens
het testen `agentRuntime.id: "codex"` voor de provider of het model in; geforceerd Codex mislukt
in plaats van terug te vallen wanneer de route of het harnas incompatibel is.

**De OpenAI Codex-runtime valt terug op het API-sleutelpad:** verzamel een geredigeerd
Gateway-fragment waarin het model, de runtime, de geselecteerde provider en de
fout worden weergegeven. Vraag betrokken medewerkers deze alleen-lezenopdracht uit te voeren op hun
OpenClaw-host:

```bash
(
  pattern='openai/gpt-5\.[45]|openai[-]codex|agentRuntime(\.id)?|harnessRuntime|Runtime: OpenAI Codex|legacy OpenAI Codex prefix|resolveSelectedOpenAIRuntimeProvider|candidateProvider[": ]+openai|status[": ]+401|Incorrect API key|No API key|api-key path|API-key path|OAuth'

  if ls /tmp/openclaw/openclaw-*.log >/dev/null 2>&1; then
    grep -E -i -n "$pattern" /tmp/openclaw/openclaw-*.log 2>/dev/null || true
  else
    journalctl --user -u openclaw-gateway --since today --no-pager 2>/dev/null \
      | grep -E -i "$pattern" || true
  fi
) | sed -E \
    -e 's/(Authorization: Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(api[_ -]?key[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/(OPENAI_API_KEY[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/sk-[A-Za-z0-9_-]{12,}/sk-[REDACTED]/g' \
    -e 's/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/[EMAIL-REDACTED]/g' \
  | tail -200
```

Nuttige fragmenten bevatten doorgaans `openai/gpt-5.6-sol` of `openai/gpt-5.6-luna`,
`Runtime: OpenAI Codex`, `agentRuntime.id` of `harnessRuntime`,
`candidateProvider: "openai"` en een resultaat van `401`, `Incorrect API key` of
`No API key`. Een gecorrigeerde uitvoering moet het OpenAI OAuth-pad tonen
in plaats van een gewone fout met een OpenAI API-sleutel.

**Configuratie met verouderde Codex-modelreferenties blijft bestaan:** voer `openclaw doctor --fix` uit.
Doctor herschrijft verouderde modelreferenties naar `openai/*`, verwijdert verouderde runtimepinnen voor sessies en
volledige agents, en behoudt bestaande overschrijvingen van authenticatieprofielen.

**De app-server wordt geweigerd:** gebruik een stabiele Codex-app-server uit `0.143.0`
via de gebundelde `0.145.0`. Prereleases, versies met een buildsuffix en nieuwere,
niet-gevalideerde releases worden geweigerd omdat OpenClaw gegenereerde schema's valideert
tegen de gebundelde app-serverversie.

**`/codex status` kan geen verbinding maken:** controleer of de Plugin `codex`
is ingeschakeld, of `plugins.allow` deze bevat wanneer een toelatingslijst is
geconfigureerd, en of eventuele aangepaste `appServer.command`, `url`, `authToken` of
headers geldig zijn.

**De Codex-app-server gebruikt te veel geheugen:** maak eerst onderscheid tussen de twee processen.
OpenClaw voert de lokale Codex-app-server uit als een afzonderlijk Rust-subproces.
`NODE_OPTIONS=--max-old-space-size=...` wijzigt alleen de Node.js V8-heap
van de Gateway; hiermee wordt Codex niet begrensd of vergroot. Beheerde Gateway-installaties kiezen al
een adaptieve V8-heap, en verhoging ervan kan minder hostgeheugen voor Codex overlaten. Gebruik
[Probleemoplossing voor Gateway-geheugen](/nl/gateway/troubleshooting#gateway-exits-during-high-memory-use)
voor geheugendruk op de Gateway en controleer het host- of containergeheugen voor het Codex-subproces.

De gebundelde Codex heeft geen heap- of RSS-limiet en geen configureerbare vertraging
voor ontladen bij inactiviteit. Nadat de laatste client zich heeft afgemeld, kan een inactieve thread
maximaal 30 minuten geladen blijven. Verminder op hosts met beperkte middelen de fan-out van systeemeigen Codex-subagents
voordat je de Gateway-heap vergroot:

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            args: ["-c", "agents.max_threads=3", "app-server", "--listen", "stdio://"],
          },
        },
      },
    },
  },
}
```

Die instelling beperkt systeemeigen childthreads voor de standaard gebundelde
multi-agent-backend van Codex. Als je Codex multi-agent v2 expliciet inschakelt, gebruik je in plaats daarvan
`features.multi_agent_v2.max_concurrent_threads_per_session=3`; de v2-
limiet omvat de rootthread en kan niet worden gecombineerd met `agents.max_threads`.
Vergroot voor meer speelruimte voor Codex de geheugentoewijzing van de host, container of cgroup.
Een harde limiet van het besturingssysteem kan Codex beëindigen in plaats van tegendruk uit te oefenen.

**Modeldetectie is traag:** verlaag
`plugins.entries.codex.config.discovery.timeoutMs` of schakel detectie uit.
Zie [Naslaginformatie voor de Codex-harness](/nl/plugins/codex-harness-reference#model-discovery).

**WebSocket-transport mislukt onmiddellijk:** controleer `appServer.url`,
`authToken`, de headers en of de externe app-server dezelfde versie van het
Codex-app-serverprotocol gebruikt. Codex WebSocket-transport blijft experimenteel
en wordt niet ondersteund; geef de voorkeur aan beheerde stdio of de lokale Unix-besturingssocket.

**Systeemeigen shell- of patchtools worden geblokkeerd met `Native hook relay
unavailable`:** de Codex-thread probeert nog steeds een systeemeigen hook-relay-
id te gebruiken dat niet meer bij OpenClaw is geregistreerd. Dit is een probleem met het
transport van systeemeigen Codex-hooks, niet met een ACP-backend, provider, GitHub of shellopdracht.
Start een nieuwe sessie in de betreffende chat met `/new` of `/reset`
en probeer daarna opnieuw een onschadelijke opdracht. Als dat eenmaal werkt, maar de volgende systeemeigen toolaanroep
opnieuw mislukt, behandel `/new` dan alleen als tijdelijke oplossing: kopieer de
prompt naar een nieuwe sessie nadat je de Codex-app-server of
OpenClaw Gateway opnieuw hebt gestart, zodat oude threads worden verwijderd en registraties van systeemeigen hooks
opnieuw worden aangemaakt.

**Codex-toolaanroepen maken te veel kortlevende hookprocessen:** stel
`plugins.entries.codex.config.appServer.loopDetectionPreToolUseRelay: false` in
en start de Gateway opnieuw. Hiermee wordt alleen het Codex-subproces `PreToolUse`
uitgeschakeld dat wordt gebruikt voor lusdetectie van OpenClaw en de bijbehorende markering voor ontbrekend beleid. Vereiste
`before_tool_call` en beleidsrelays voor vertrouwde tools blijven ingeschakeld.

**Een niet-Codex-model gebruikt de ingebouwde harness:** dit is te verwachten, tenzij beleid voor de
provider- of modelruntime het naar een andere harness routeert. Gewone referenties van niet-OpenAI-
providers blijven in de modus `auto` hun normale providerpad gebruiken.

**Computer Use is geïnstalleerd, maar tools worden niet uitgevoerd:** controleer
`/codex computer-use status` vanuit een nieuwe sessie. Als een tool
`Native hook relay unavailable` meldt, gebruik dan het bovenstaande herstel voor de systeemeigen hook-relay.
Zie [Codex Computer Use](/nl/plugins/codex-computer-use#troubleshooting).

## Gerelateerd

- [Naslaginformatie voor de Codex-harness](/nl/plugins/codex-harness-reference)
- [Runtime van de Codex-harness](/nl/plugins/codex-harness-runtime)
- [Codex-supervisie](/plugins/codex-supervision)
- [Systeemeigen Codex-plugins](/nl/plugins/codex-native-plugins)
- [Codex Computer Use](/nl/plugins/codex-computer-use)
- [Agentruntimes](/nl/concepts/agent-runtimes)
- [Modelproviders](/nl/concepts/model-providers)
- [OpenAI-provider](/nl/providers/openai)
- [Hulp voor OpenAI Codex](https://help.openai.com/en/collections/14937394-codex)
- [Plugins voor agentharnassen](/nl/plugins/sdk-agent-harness)
- [Plugin-hooks](/nl/plugins/hooks)
- [Diagnostische gegevens exporteren](/nl/gateway/diagnostics)
- [Status](/nl/cli/status)
- [Testen](/nl/help/testing-live#live-codex-app-server-harness-smoke)
