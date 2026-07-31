---
read_when:
    - Het aanpassen van de parsing of standaardwaarden voor thinking-, fast-mode- of verbose-directieven
summary: Directivesyntaxis voor /think, /fast, /verbose, /trace en zichtbaarheid van de redenering
title: Denkniveaus
x-i18n:
    generated_at: "2026-07-27T05:28:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80968ce58f642090ba0f807874e43eea1206cd31d919414c690b7537dc523658
    source_path: tools/thinking.md
    workflow: 16
---

## Wat het doet

- Inline-instructie in de tekst van elk inkomend bericht: `/t <level>`, `/think:<level>` of `/thinking <level>`.
- Niveaus (aliassen): `off | minimal | low | medium | high | xhigh | adaptive | max | ultra`, grofweg overeenkomstig de klassieke reeks magische woorden van Anthropic: "think" < "think hard" < "think harder" < "ultrathink":
  - minimal ~ "think"
  - low ~ "think hard"
  - medium ~ "think harder"
  - high ~ "ultrathink" (maximaal budget)
  - xhigh ~ "ultrathink+" (GPT-5.2+- en Codex-modellen, plus inspanningsniveau van Anthropic Claude Opus 4.7+)
  - adaptive → adaptief denken dat door de provider wordt beheerd (ondersteund voor Claude 4.6 op Anthropic/Bedrock, Anthropic Claude Opus 4.7+ en dynamisch denken van Google Gemini)
  - max → maximale redeneerinspanning van de provider (Anthropic Claude Opus 4.7+; Ollama koppelt dit aan het hoogste systeemeigen `think`-inspanningsniveau)
  - ultra → maximale redeneerinspanning van de provider plus proactieve subagentorkestratie wanneer het geselecteerde model/de geselecteerde runtime dit ondersteunt
  - `x-high`, `x_high`, `extra-high`, `extra high` en `extra_high` worden gekoppeld aan `xhigh`.
  - `highest` wordt gekoppeld aan `high`.
- Opmerkingen over providers:
  - Denkmenu's en keuzelijsten worden aangestuurd door providerprofielen. Providerplugins declareren de exacte reeks niveaus voor het geselecteerde model, inclusief labels zoals het binaire `on`.
  - `adaptive`, `xhigh`, `max` en `ultra` worden alleen aangeboden voor provider-/model-/runtimeprofielen die ze ondersteunen. Getypte instructies voor niet-ondersteunde niveaus worden geweigerd met de geldige opties van dat model.
  - Bestaande opgeslagen niet-ondersteunde niveaus worden opnieuw gekoppeld op basis van de rangorde van het providerprofiel. `adaptive` valt bij niet-adaptieve modellen terug op `medium`, terwijl `xhigh` en `max` terugvallen op het hoogste ondersteunde niveau dat niet uitgeschakeld is voor het geselecteerde model.
  - Anthropic Claude 4.6-modellen gebruiken standaard `adaptive` wanneer er geen expliciet denkniveau is ingesteld.
  - Bij Anthropic Claude Opus 4.8 en Opus 4.7 blijft denken uitgeschakeld, tenzij je expliciet een denkniveau instelt. Nadat adaptief denken is ingeschakeld, is de door de provider beheerde standaardinspanning van Opus 4.8 `high`.
  - Anthropic Claude Opus 4.7+ koppelt `/think xhigh` aan adaptief denken plus `output_config.effort: "xhigh"`, omdat `/think` een denkinstructie is en `xhigh` de inspanningsinstelling van Opus is.
  - Anthropic Claude Opus 4.7+ biedt ook `/think max`; dit wordt aan hetzelfde door de provider beheerde pad voor maximale inspanning gekoppeld.
  - Directe DeepSeek V4-modellen bieden `/think xhigh|max`; beide worden gekoppeld aan DeepSeek `reasoning_effort: "max"`, terwijl lagere niveaus die niet uitgeschakeld zijn aan `high` worden gekoppeld.
  - Via OpenRouter gerouteerde DeepSeek V4-modellen bieden `/think xhigh` en verzenden door OpenRouter ondersteunde `reasoning.effort`-waarden in plaats van het systeemeigen DeepSeek-topniveau `reasoning_effort`. Lagere niveaus die niet uitgeschakeld zijn, worden gekoppeld aan `high` en opgeslagen `max`-overschrijvingen vallen terug op `xhigh`.
  - Ollama-modellen die denken ondersteunen, bieden `/think low|medium|high|max`; `max` wordt gekoppeld aan het systeemeigen `think: "high"`, omdat de systeemeigen API van Ollama de inspanningswaarden `low`, `medium` en `high` accepteert.
  - OpenAI GPT-modellen koppelen `/think` via modelspecifieke ondersteuning voor inspanningsniveaus in de Responses API. `/think off` verzendt `reasoning.effort: "none"` alleen wanneer het doelmodel dit ondersteunt; anders laat OpenClaw de uitgeschakelde redeneerpayload weg in plaats van een niet-ondersteunde waarde te verzenden.
  - GPT-5.6 Sol en Terra bieden het systeemeigen `/think ultra` via de Codex-runtime. GPT-5.6 Luna biedt niveaus tot en met `max`, omdat de Codex-catalogus daarvan Ultra niet vermeldt.
  - De ingebedde OpenClaw-runtime biedt logisch `/think ultra` voor GPT-5.6 Sol, Terra en Luna. Deze verzendt de maximale inspanning van de provider en voegt richtlijnen toe voor proactieve subagentorkestratie binnen de uitvoering.
  - Aangepaste OpenAI-compatibele catalogusvermeldingen kunnen `/think xhigh` inschakelen door `models.providers.<provider>.models[].compat.supportedReasoningEfforts` zo in te stellen dat deze `"xhigh"` bevat. Hiervoor worden dezelfde compatibiliteitsmetadata gebruikt waarmee uitgaande OpenAI-payloads voor redeneerinspanning worden gekoppeld, zodat menu's, sessievalidatie, de agent-CLI en `llm-task` overeenkomen met het transportgedrag.
  - Verouderde geconfigureerde verwijzingen naar OpenRouter Hunter Alpha slaan de injectie van proxyredenering over, omdat die ingetrokken route definitieve antwoordtekst via redeneervelden kon retourneren.
  - Google Gemini koppelt `/think adaptive` aan het door Gemini's provider beheerde dynamische denken. Gemini 3-aanvragen laten een vaste `thinkingLevel` weg, terwijl Gemini 2.5-aanvragen `thinkingBudget: -1` verzenden; vaste niveaus worden nog steeds gekoppeld aan het dichtstbijzijnde Gemini-`thinkingLevel` of budget voor die modelfamilie.
  - MiniMax M2.x (`minimax/MiniMax-M2*`) op het Anthropic-compatibele streamingpad gebruikt standaard `thinking: { type: "disabled" }`, tenzij je denken expliciet instelt in modelparameters of aanvraagparameters. Dit voorkomt dat `reasoning_content`-delta's uit de niet-systeemeigen Anthropic-streamindeling van M2.x uitlekken. MiniMax-M3 (en M3.x) is uitgezonderd: M3 produceert correcte Anthropic-denkblokken en retourneert lege inhoud wanneer denken is uitgeschakeld. Daarom houdt OpenClaw M3 op het door de provider beheerde pad voor weggelaten/adaptief denken.
  - Z.AI (`zai/*`) is voor de meeste GLM-modellen binair (`on`/`off`). GLM-5.2 is de uitzondering: dit model biedt `/think off|low|high|max`, koppelt `low` en `high` aan Z.AI `reasoning_effort: "high"` en koppelt `max` aan `reasoning_effort: "max"`.
  - Moonshot API Kimi K3 (`moonshot/kimi-k3`) denkt altijd op `max`, verzendt `reasoning_effort: "max"`, laat het K2-veld `thinking` en vaste steekproefoverschrijvingen weg en behoudt de door K3 ondersteunde gereedschapskeuzes. Kimi Code K3 (`kimi/k3` en `kimi/k3[1m]`) biedt `/think off|max`: uitgeschakeld verzendt `thinking.type: "disabled"`, terwijl maximaal adaptief denken met maximale inspanning verzendt. Huidige Kimi Code-verwijzingen bevatten ook `kimi/kimi-for-coding` en `kimi/kimi-for-coding-highspeed`. Kimi K2.7 Code (`moonshot/kimi-k2.7-code` en `moonshot/kimi-k2.7-code-highspeed`) denkt altijd, biedt alleen `on` en laat zowel uitgaand `thinking` als `reasoning_effort` weg. Andere `moonshot/*`-modellen koppelen `/think off` aan `thinking: { type: "disabled" }` en elk niveau anders dan `off` aan `thinking: { type: "enabled" }`. Wanneer K2-denken is ingeschakeld, accepteert Moonshot alleen `tool_choice` `auto|none`; OpenClaw normaliseert incompatibele waarden naar `auto`.

## Volgorde van bepaling

1. Inline-instructie in het bericht (geldt alleen voor dat bericht).
2. Sessieoverschrijving (ingesteld door een bericht te verzenden dat alleen een instructie bevat).
3. Standaard per agent (`agents.entries.*.thinkingDefault` in de configuratie).
4. Globale standaard (`agents.defaults.thinkingDefault` in de configuratie).
5. Terugval: door de provider gedeclareerde standaard indien beschikbaar; anders worden modellen met redeneervermogen ingesteld op `medium` of het dichtstbijzijnde ondersteunde niveau anders dan `off` voor dat model, en blijven modellen zonder redeneervermogen `off`.

## Een sessiestandaard instellen

- Verzend een bericht dat **alleen** de instructie bevat (witruimte is toegestaan), bijvoorbeeld `/think:medium` of `/t high`.
- Dit blijft gelden voor de huidige sessie (standaard per afzender). Gebruik `/think default` om de sessieoverschrijving te wissen en de geconfigureerde standaard of providerstandaard over te nemen; aliassen zijn onder meer `inherit`, `clear`, `reset` en `unpin`.
- `/think off` slaat een expliciete overschrijving voor uitgeschakeld op. Dit schakelt denken uit totdat je de sessieoverschrijving wijzigt of wist.
- Er wordt een bevestigingsantwoord verzonden (`Thinking level set to high.` / `Thinking disabled.`). Als het niveau ongeldig is (bijvoorbeeld `/thinking big`), wordt de opdracht geweigerd met een aanwijzing en blijft de sessiestatus ongewijzigd.
- Verzend `/think` (of `/think:`) zonder argument om het huidige denkniveau te bekijken.

## Toepassing per agent

- **Ingebedde OpenClaw**: het bepaalde niveau wordt doorgegeven aan de OpenClaw-agentruntime binnen het proces.
- **Claude CLI-backend**: concrete niveaus die niet uitgeschakeld zijn, worden bij gebruik van `claude-cli` als `--effort` doorgegeven aan Claude Code; `adaptive` verwijdert geconfigureerde inspanningsvlaggen en delegeert de effectieve inspanning aan de omgeving, instellingen en modelstandaarden van Claude Code. Zie [CLI-backends](/nl/gateway/cli-backends).

## Snelle modus (/fast)

- Niveaus: `auto|on|off|default`.
- Een bericht dat alleen een instructie bevat, schakelt een sessieoverschrijving voor de snelle modus in of uit en antwoordt met `Fast mode set to auto.`, `Fast mode enabled.` of `Fast mode disabled.`. Gebruik `/fast default` om de sessieoverschrijving te wissen en de geconfigureerde standaard over te nemen; aliassen zijn onder meer `inherit`, `clear`, `reset` en `unpin`.
- Verzend `/fast` (of `/fast status`) zonder modus om de huidige effectieve status van de snelle modus te bekijken.
- OpenClaw bepaalt de snelle modus in deze volgorde:
  1. Inline-/alleen-instructieoverschrijving via `/fast auto|on|off` (`/fast default` wist deze laag)
  2. Sessieoverschrijving
  3. Standaard per agent (`agents.entries.*.fastModeDefault`)
  4. Configuratie per model: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. Terugval: `off`
- `auto` houdt de sessie-/configuratiemodus op automatisch, maar bepaalt elke nieuwe modelaanroep afzonderlijk. Aanroepen die vóór de automatische afkapgrens beginnen, hebben de snelle modus ingeschakeld; latere nieuwe pogingen, terugvallen, gereedschapsresultaten of vervolgaanroepen beginnen met uitgeschakelde snelle modus. De afkapgrens is standaard 60 seconden; stel `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` in op het actieve model om deze te wijzigen.
- Voor `openai/*` wordt de snelle modus gekoppeld aan prioriteitsverwerking van OpenAI door `service_tier=priority` te verzenden bij ondersteunde Responses-aanvragen.
- Voor door Codex ondersteunde `openai/*`- / `openai-codex/*`-modellen verzendt de snelle modus dezelfde `service_tier=priority`-vlag bij Codex Responses. Systeemeigen beurten van de Codex-app-server ontvangen het niveau alleen bij `turn/start` of bij het starten/hervatten van een thread. Daardoor kan `auto` het niveau van een reeds actieve app-serverbeurt niet wijzigen; het geldt voor de volgende modelbeurt die OpenClaw start.
- Voor directe openbare `anthropic/*`-aanvragen, inclusief met OAuth geverifieerd verkeer dat naar `api.anthropic.com` wordt verzonden, wordt de snelle modus gekoppeld aan Anthropic-serviceniveaus: `/fast on` stelt `service_tier=auto` in, `/fast off` stelt `service_tier=standard_only` in.
- Voor `minimax/*` op het Anthropic-compatibele pad herschrijft `/fast on` (of `params.fastMode: true`) `MiniMax-M2.7` naar `MiniMax-M2.7-highspeed`.
- Expliciete Anthropic-modelparameters `serviceTier` / `service_tier` overschrijven de standaard van de snelle modus wanneer beide zijn ingesteld. OpenClaw slaat de injectie van Anthropic-serviceniveaus nog steeds over voor niet-Anthropic-basis-URL's van proxy's.
- `/status` toont `Fast` wanneer de snelle modus is ingeschakeld en `Fast:auto` wanneer de geconfigureerde modus automatisch is.

## Uitgebreide instructies (/verbose of /v)

- Niveaus: `on` (minimaal) | `full` | `off` (standaard).
- Een bericht met alleen een richtlijn schakelt uitgebreide sessie-uitvoer in of uit en antwoordt met `Verbose logging enabled.` / `Verbose logging disabled.`; ongeldige niveaus geven een hint zonder de status te wijzigen.
- `/verbose off` slaat een expliciete sessie-overschrijving op; wis deze via de sessie-interface door `inherit` te kiezen.
- Geautoriseerde afzenders van externe kanalen mogen de overschrijving voor uitgebreide sessie-uitvoer permanent opslaan. Interne Gateway-/webchatclients hebben `operator.admin` nodig om deze permanent op te slaan.
- Een inline richtlijn is alleen van invloed op dat bericht; anders gelden de sessie-/globale standaardwaarden.
- Stuur `/verbose` (of `/verbose:`) zonder argument om het huidige niveau voor uitgebreide uitvoer te bekijken.
- Wanneer uitgebreide uitvoer is ingeschakeld, sturen agents die gestructureerde toolresultaten genereren elke toolaanroep terug als een afzonderlijk bericht met alleen metagegevens, indien beschikbaar voorafgegaan door `<emoji> <tool-name>: <arg>`. Deze toolsamenvattingen worden verzonden zodra elke tool wordt gestart (afzonderlijke tekstballonnen), niet als streamende delta's.
- Samenvattingen van toolfouten blijven zichtbaar in de normale modus, maar achtervoegsels met onbewerkte foutdetails worden verborgen, tenzij uitgebreide uitvoer `full` is.
- Wanneer uitgebreide uitvoer `full` is, worden tooluitvoerresultaten na voltooiing ook doorgestuurd (afzonderlijke tekstballon, afgekapt tot een veilige lengte). Als je `/verbose on|full|off` omschakelt terwijl een uitvoering bezig is, volgen volgende tooltekstballonnen de nieuwe instelling.
- `agents.defaults.toolProgressDetail` bepaalt de vorm van `/verbose`-toolsamenvattingen en toolregels in voortgangsconcepten. Gebruik `"explain"` (standaard) voor compacte, voor mensen leesbare labels zoals `🛠️ Exec: checking JS syntax`; gebruik `"raw"` als je voor foutopsporing ook de onbewerkte opdracht/details wilt toevoegen. `agents.entries.*.toolProgressDetail` per agent overschrijft de standaardwaarde.
  - `explain`: `🛠️ Exec: check JS syntax for /tmp/app.js`
  - `raw`: `🛠️ Exec: check JS syntax for /tmp/app.js, node --check /tmp/app.js`

## Plugin-traceringsrichtlijnen (/trace)

- Niveaus: `on` | `off` (standaard).
- Een bericht met alleen een richtlijn schakelt de Plugin-traceringsuitvoer voor de sessie in of uit en antwoordt met `Plugin trace enabled.` / `Plugin trace disabled.`.
- Een inline richtlijn is alleen van invloed op dat bericht; anders gelden de sessie-/globale standaardwaarden.
- Stuur `/trace` (of `/trace:`) zonder argument om het huidige traceringsniveau te bekijken.
- `/trace` is beperkter dan `/verbose`: het toont alleen tracerings-/foutopsporingsregels die eigendom zijn van de Plugin, zoals foutopsporingssamenvattingen van Active Memory.
- Traceringsregels kunnen verschijnen in `/status` en als diagnostisch vervolgbericht na het normale antwoord van de assistent.

## Zichtbaarheid van redeneringen (/reasoning)

- Niveaus: `on|off|stream`.
- Een bericht met alleen een richtlijn schakelt in of denkblokken in antwoorden worden weergegeven.
- Wanneer dit is ingeschakeld, wordt de redenering verzonden als een **afzonderlijk bericht**, voorafgegaan door `Thinking`.
- `stream`: streamt de redenering terwijl het antwoord wordt gegenereerd wanneer het actieve kanaal voorbeelden van redeneringen ondersteunt en verzendt daarna het definitieve antwoord zonder redenering.
- Alias: `/reason`.
- Stuur `/reasoning` (of `/reasoning:`) zonder argument om het huidige redeneerniveau te bekijken.
- Volgorde van resolutie: inline richtlijn, vervolgens sessie-overschrijving, vervolgens standaardwaarde per agent (`agents.entries.*.reasoningDefault`), vervolgens globale standaardwaarde (`agents.defaults.reasoningDefault`) en ten slotte terugvalwaarde (`off`).

Ongeldige redeneringstags van lokale modellen worden terughoudend verwerkt. Gesloten `<think>...</think>`-blokken blijven verborgen in normale antwoorden, en een niet-gesloten redenering na reeds zichtbare tekst wordt eveneens verborgen. Als een antwoord volledig is omgeven door één niet-gesloten openingstag en anders als lege tekst zou worden afgeleverd, verwijdert OpenClaw de ongeldige openingstag en levert het de resterende tekst af.

## Gerelateerd

- Documentatie over de verhoogde modus staat in [Verhoogde modus](/nl/tools/elevated).

## Heartbeats

- De inhoud van de Heartbeat-probe is de geconfigureerde Heartbeat-prompt (standaard: `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`). Inline richtlijnen in een Heartbeat-bericht worden zoals gebruikelijk toegepast (maar voorkom dat Heartbeats de standaardwaarden van de sessie wijzigen).
- Heartbeat-bezorging gebruikt standaard alleen de definitieve payload. Stel `agents.defaults.heartbeat.includeReasoning: true` of `agents.entries.*.heartbeat.includeReasoning: true` per agent in om ook het afzonderlijke `Thinking`-bericht te verzenden (indien beschikbaar).

## Webchatinterface

- Wanneer de pagina wordt geladen, weerspiegelt de denkniveaukiezer van de webchat het opgeslagen niveau van de sessie uit de opslag/configuratie van de inkomende sessie.
- Als je een ander niveau kiest, wordt de sessie-overschrijving onmiddellijk via `sessions.patch` opgeslagen; er wordt niet gewacht op de volgende verzending en het is geen eenmalige `thinkingOnce`-overschrijving.
- Bij verzenden terwijl wijzigingen in de model-, redeneer- of snelheidskiezer nog worden toegepast, wordt gewacht op elke openstaande kiezerpatch; als een wijziging mislukt, blijft het bericht onverzonden ter controle.
- De eerste optie is altijd de keuze om de overschrijving te wissen. Deze toont `Inherited: <resolved level>`, inclusief `Inherited: Off` wanneer overgenomen denken is uitgeschakeld.
- Expliciete keuzes in de kiezer gebruiken hun directe niveaulabels, waarbij providerlabels behouden blijven als die aanwezig zijn (bijvoorbeeld `Maximum` voor een door de provider gelabelde `max`-optie).
- De kiezer gebruikt `thinkingLevels` die door de Gateway-sessierij/-standaardwaarden worden geretourneerd, waarbij `thinkingOptions` behouden blijft als verouderde labellijst. De browserinterface houdt geen eigen lijst met reguliere expressies voor providers bij; plugins beheren modelspecifieke niveausets.
- `/think:<level>` blijft werken en werkt hetzelfde opgeslagen sessieniveau bij, zodat chatrichtlijnen en de kiezer gesynchroniseerd blijven.

## Providerprofielen

- Providerplugins kunnen `resolveThinkingProfile(ctx)` beschikbaar stellen om de ondersteunde niveaus en standaardwaarde van het model te definiëren.
- Providerplugins die Claude-modellen als proxy doorgeven, moeten `resolveClaudeThinkingProfile(modelId)` uit `openclaw/plugin-sdk/provider-model-shared` hergebruiken, zodat directe Anthropic-catalogi en proxycatalogi op elkaar afgestemd blijven.
- Elk profielniveau heeft een opgeslagen canonieke `id` (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max` of `ultra`) en kan een `label` voor weergave bevatten. Binaire providers gebruiken `{ id: "low", label: "on" }`.
- Profielhooks ontvangen samengevoegde catalogusgegevens wanneer die beschikbaar zijn, waaronder `reasoning`, `compat.thinkingFormat` en `compat.supportedReasoningEfforts`. Gebruik die gegevens om binaire of aangepaste profielen alleen beschikbaar te stellen wanneer het geconfigureerde aanvraagcontract de bijbehorende payload ondersteunt.
- Toolplugins die een expliciete overschrijving van het denkniveau moeten valideren, moeten `api.runtime.agent.resolveThinkingPolicy({ provider, model, agentRuntime })` samen met `api.runtime.agent.normalizeThinkingLevel(...)` gebruiken; ze moeten geen eigen lijsten met provider-/modelniveaus bijhouden. Geef `agentRuntime` door wanneer de tool het uitvoeringspad beheert, zoals bij een altijd ingesloten uitvoering.
- Toolplugins met toegang tot geconfigureerde aangepaste modelmetagegevens kunnen `catalog` doorgeven aan `resolveThinkingPolicy`, zodat `compat.supportedReasoningEfforts`-aanmeldingen worden meegenomen in validatie aan de Plugin-zijde.
- Gepubliceerde verouderde hooks (`supportsXHighThinking`, `isBinaryThinking` en `resolveDefaultThinkingLevel`) blijven beschikbaar als compatibiliteitsadapters, maar nieuwe aangepaste niveausets moeten `resolveThinkingProfile` gebruiken.
- Gateway-rijen/-standaardwaarden stellen `thinkingLevels`, `thinkingOptions` en `thinkingDefault` beschikbaar, zodat ACP-/chatclients dezelfde profiel-ID's en labels weergeven als door de runtimevalidatie worden gebruikt.
