---
read_when:
    - Standaardinstellingen voor agents afstemmen (modellen, denkproces, werkruimte, heartbeat, media, skills)
    - Routering en bindingen voor meerdere agents configureren
    - Sessie-, berichtbezorgings- en gespreksmodusgedrag aanpassen
summary: Standaardinstellingen voor agents, multi-agentroutering, sessie-, berichten- en spraakconfiguratie
title: Configuratie — agents
x-i18n:
    generated_at: "2026-07-27T05:50:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7a161d65b02e3333c15a2d998421419ee37d36be4d02ebb3a86e66282df06adb
    source_path: gateway/config-agents.md
    workflow: 16
---

Configuratiesleutels met agentbereik onder `agents.*`, `multiAgent.*`, `session.*`,
`messages.*` en `talk.*`. Zie voor kanalen, tools, de Gateway-runtime en andere
sleutels op het hoogste niveau de [Configuratiereferentie](/nl/gateway/configuration-reference).

## Standaardinstellingen voor agents

### `agents.defaults.workspace`

Standaard: `OPENCLAW_WORKSPACE_DIR` indien ingesteld, anders `~/.openclaw/workspace` (of `~/.openclaw/workspace-<profile>` wanneer `OPENCLAW_PROFILE` is ingesteld op een niet-standaardprofiel).

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

Een expliciete waarde voor `agents.defaults.workspace` heeft voorrang op
`OPENCLAW_WORKSPACE_DIR`. Gebruik de omgevingsvariabele om standaardagents
naar een gekoppelde werkruimte te laten verwijzen wanneer je dat pad niet in de configuratie wilt opnemen.

### `agents.defaults.repoRoot`

Optionele hoofdmap van de repository die wordt weergegeven in de Runtime-regel van de systeemprompt. Indien niet ingesteld, detecteert OpenClaw deze automatisch door vanaf de werkruimte omhoog te navigeren.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Optionele standaardtoelatingslijst voor Skills voor agents die
`agents.entries.*.skills` niet instellen.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // neemt github, weather over
      { id: "docs", skills: ["docs-search"] }, // vervangt standaardinstellingen
      { id: "locked-down", skills: [] }, // geen Skills
    ],
  },
}
```

- Laat `agents.defaults.skills` weg om standaard onbeperkte Skills toe te staan.
- Laat `agents.entries.*.skills` weg om de standaardinstellingen over te nemen.
- Stel `agents.entries.*.skills: []` in voor geen Skills.
- Een niet-lege lijst voor `agents.entries.*.skills` is de definitieve set voor die agent; deze
  wordt niet samengevoegd met de standaardinstellingen.

### `agents.defaults.skipBootstrap`

Schakelt het automatisch aanmaken van bootstrapbestanden voor de werkruimte uit (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.skipOptionalBootstrapFiles`

Slaat het aanmaken van geselecteerde optionele werkruimtebestanden over, terwijl vereiste bootstrapbestanden (`AGENTS.md`, `TOOLS.md`, `BOOTSTRAP.md`) nog steeds worden geschreven. Geldige waarden: `SOUL.md`, `USER.md` en `IDENTITY.md` (`HEARTBEAT.md` wordt geaccepteerd, maar doet niets omdat de Heartbeat-context naar de tijdelijke opslag van de Cron-monitor is verplaatst).

```json5
{
  agents: {
    defaults: {
      skipOptionalBootstrapFiles: ["SOUL.md", "USER.md"],
    },
  },
}
```

### `agents.defaults.contextInjection`

Bepaalt wanneer bootstrapbestanden van de werkruimte in de systeemprompt worden geïnjecteerd. Standaard: `"always"`.

- `"continuation-skip"`: bij veilige vervolgbeurten (na een voltooid antwoord van de assistent) wordt herinjectie van de werkruimtebootstrap overgeslagen, waardoor de prompt kleiner wordt. Heartbeat-uitvoeringen en nieuwe pogingen na Compaction bouwen de context nog steeds opnieuw op.
- `"never"`: schakelt de injectie van de werkruimtebootstrap en contextbestanden bij elke beurt uit. Gebruik dit alleen voor agents die hun promptlevenscyclus volledig zelf beheren (aangepaste contextengines, native runtimes die hun eigen context opbouwen of gespecialiseerde workflows zonder bootstrap). Bij Heartbeat- en herstelbeurten na Compaction wordt de injectie eveneens overgeslagen.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

Overschrijving per agent: `agents.entries.*.contextInjection`. Weggelaten waarden nemen
`agents.defaults.contextInjection` over.

### `agents.defaults.bootstrapMaxChars`

Maximumaantal tekens per bootstrapbestand van de werkruimte vóór afkapping. Standaard: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

Overschrijving per agent: `agents.entries.*.bootstrapMaxChars`. Weggelaten waarden nemen
`agents.defaults.bootstrapMaxChars` over.

### `agents.defaults.bootstrapTotalMaxChars`

Maximumaantal tekens dat in totaal uit alle bootstrapbestanden van de werkruimte wordt geïnjecteerd. Standaard: `60000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

Overschrijving per agent: `agents.entries.*.bootstrapTotalMaxChars`. Weggelaten waarden
nemen `agents.defaults.bootstrapTotalMaxChars` over.

### Overschrijvingen van bootstrapprofielen per agent

Gebruik overschrijvingen van bootstrapprofielen per agent wanneer één agent ander
promptinjectiegedrag nodig heeft dan de gedeelde standaardinstellingen. Weggelaten velden nemen waarden over van
`agents.defaults`.

```json5
{
  agents: {
    defaults: {
      contextInjection: "continuation-skip",
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 60000,
    },
    list: [
      {
        id: "strict-worker",
        contextInjection: "always",
        bootstrapMaxChars: 50000,
        bootstrapTotalMaxChars: 300000,
      },
    ],
  },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Bepaalt de melding in de systeemprompt die voor de agent zichtbaar is wanneer de bootstrapcontext wordt afgekapt.
Standaard: `"always"`.

- `"off"`: injecteer nooit tekst met een afkappingsmelding in de systeemprompt.
- `"once"`: injecteer eenmaal per unieke afkappingshandtekening een beknopte melding.
- `"always"`: injecteer bij elke uitvoering een beknopte melding wanneer er afkapping plaatsvindt (aanbevolen).

Gedetailleerde ruwe/geïnjecteerde aantallen en velden voor configuratieafstemming blijven beschikbaar in diagnostiek, zoals
context-/statusrapporten en logboeken; de gebruikelijke gebruikers-/runtimecontext van WebChat
krijgt alleen de beknopte herstelmelding.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "always" } }, // off | once | always
}
```

### Eigendomsoverzicht voor contextbudgetten

OpenClaw heeft meerdere omvangrijke prompt-/contextbudgetten. Deze zijn
bewust per subsysteem opgesplitst in plaats van allemaal via één algemene
instelling te lopen.

| Budget                                                         | Omvat                                                                                                                                                           |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` | Normale injectie van de werkruimtebootstrap                                                                                                                     |
| `agents.defaults.startupContext.*`                             | Eenmalige prelude voor modeluitvoering bij reset/opstart, inclusief recente dagelijkse `memory/*.md`-bestanden. Kale chatopdrachten `/new` en `/reset` worden bevestigd zonder het model aan te roepen |
| `skills.limits.*`                                              | De compacte lijst met Skills die in de systeemprompt wordt geïnjecteerd                                                                                         |
| `agents.defaults.contextLimits.*`                              | Begrensde runtimefragmenten en geïnjecteerde blokken die eigendom zijn van de runtime                                                                            |
| `memory.qmd.limits.*`                                          | Grootte van geïndexeerde fragmenten voor geheugenzoekopdrachten en injectie                                                                                      |

Overeenkomstige overschrijvingen per agent:

- `agents.entries.*.skillsLimits.maxSkillsPromptChars`
- `agents.entries.*.contextInjection`
- `agents.entries.*.bootstrapMaxChars`
- `agents.entries.*.bootstrapTotalMaxChars`
- `agents.entries.*.contextLimits.*`

#### `agents.defaults.startupContext`

Beheert de opstartprelude voor de eerste beurt die bij modeluitvoeringen voor reset/opstart wordt geïnjecteerd.
Kale chatopdrachten `/new` en `/reset` bevestigen de reset zonder
het model aan te roepen en laden deze prelude daarom niet.

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

Gedeelde standaardinstellingen voor begrensde runtimecontextoppervlakken.

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}
```

- `memoryGetMaxChars`: standaardlimiet voor `memory_get`-fragmenten voordat afkappingsmetadata
  en een vervolgmelding worden toegevoegd.
- Wanneer `memory_get` geen `lines` bevat, gebruikt OpenClaw een ingebouwd venster van 120 regels en
  past vervolgens `memoryGetMaxChars` toe.
- Live toolresultaten gebruiken een automatische limiet voor de modelcontext: `16000` tekens onder 100K
  tokens, `32000` tekens bij 100K+ tokens en `64000` tekens bij 200K+ tokens.
- `postCompactionMaxChars`: limiet voor het AGENTS.md-fragment dat wordt gebruikt tijdens
  vernieuwingsinjectie na Compaction.

#### `agents.entries.*.contextLimits`

Overschrijving per agent voor de gedeelde `contextLimits`-instellingen. Weggelaten velden nemen waarden over
van `agents.defaults.contextLimits`.

```json5
{
  agents: {
    defaults: {
      contextLimits: { memoryGetMaxChars: 12000 },
    },
    list: [
      {
        id: "tiny-local",
        contextLimits: {
          memoryGetMaxChars: 6000,
        },
      },
    ],
  },
}
```

#### `skills.limits.maxSkillsPromptChars`

Algemene limiet voor de compacte lijst met Skills die in de systeemprompt wordt geïnjecteerd. Dit
heeft geen invloed op het op aanvraag lezen van `SKILL.md`-bestanden.

```json5
{
  skills: { limits: { maxSkillsPromptChars: 18000 } },
}
```

#### `agents.entries.*.skillsLimits.maxSkillsPromptChars`

Overschrijving per agent voor het promptbudget voor Skills.

```json5
{
  agents: {
    list: [{ id: "tiny-local", skillsLimits: { maxSkillsPromptChars: 6000 } }],
  },
}
```

### `agents.defaults.imageMaxDimensionPx`

Maximale pixelafmeting voor de langste zijde van afbeeldingen in transcript-/toolafbeeldingsblokken vóór provideraanroepen.
Standaard: `1200`.

Lagere waarden verminderen meestal het gebruik van visietokens en de grootte van aanvraagpayloads bij uitvoeringen met veel schermafbeeldingen.
Hogere waarden behouden meer visuele details.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.imageQuality`

Voorkeur voor compressie/detail van de afbeeldingstool voor afbeeldingen die uit bestandspaden, URL's en mediaverwijzingen worden geladen.
Standaard: `auto`.

OpenClaw past de schaalstappen aan het geselecteerde afbeeldingsmodel aan. Claude Opus 4.8, OpenAI GPT-5.6 Sol, Qwen VL en gehoste Llama 4-visiemodellen kunnen bijvoorbeeld grotere afbeeldingen gebruiken dan oudere/standaard visiepaden met veel detail, terwijl beurten met meerdere afbeeldingen in de modus `auto` agressiever worden gecomprimeerd om de kosten voor tokens en latentie te beheersen.

Waarden:

- `auto`: aanpassen aan modellimieten en het aantal afbeeldingen.
- `efficient`: kleinere afbeeldingen verkiezen voor lager token- en bytegebruik.
- `balanced`: de standaard schaalstappen als middenweg gebruiken.
- `high`: meer details behouden voor schermafbeeldingen, diagrammen en documentafbeeldingen.

```json5
{
  agents: { defaults: { imageQuality: "auto" } },
}
```

### `agents.defaults.userTimezone`

Tijdzone voor context in de systeemprompt (niet voor berichttijdstempels). Valt terug op de tijdzone van de host.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Tijdnotatie in de systeemprompt. Standaard: `auto` (voorkeur van het besturingssysteem).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      utilityModel: "openai/gpt-5.4-mini",
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      mediaModels: {
        image: {
          primary: "openai/gpt-image-2",
          fallbacks: ["google/gemini-3.1-flash-image"],
        },
        video: {
          primary: "qwen/wan2.6-t2v",
          fallbacks: ["qwen/wan2.6-i2v"],
        },
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // algemene standaardparameters voor de provider
      pdfMaxMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      toolProgressDetail: "explain",
      reasoningDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 4,
    },
  },
}
```

- `model`: accepteert een tekenreeks (`"provider/model"`) of een object (`{ primary, fallbacks }`).
  - De tekenreeksvorm stelt alleen het primaire model in.
  - De objectvorm stelt het primaire model plus geordende failovermodellen in.
- `utilityModel`: optionele `provider/model`-verwijzing of alias voor korte interne taken. Deze wordt momenteel gebruikt voor gegenereerde sessietitels in de Control UI, onderwerptitels voor Telegram-DM's, automatische threadtitels in Discord en [vertelling bij voortgangsconcepten](/nl/concepts/progress-drafts#narrated-status). Wanneer deze niet is ingesteld, leidt OpenClaw de door de primaire provider opgegeven standaard voor kleine modellen af als die bestaat (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`); anders gebruiken titeltaken het primaire model van de agent en blijft vertelling uitgeschakeld. Als een afzonderlijk hulpmiddelmodel een gegenereerde titel niet kan voorbereiden of voltooien, probeert OpenClaw die titel eenmaal opnieuw met het primaire model. Voor dashboardtitels gebruiken automatische afleiding van het hulpmiddelmodel en de normale terugvaloptie de effectieve sessieprovider en het authenticatieprofiel; een expliciet hulpmiddelmodel behoudt de geconfigureerde provider/authenticatie. Stel `utilityModel: ""` in om de alternatieve hulpmiddelroute over te slaan; het genereren van dashboardtitels gaat dan nog steeds rechtstreeks verder met het normale sessiemodel. `agents.entries.*.utilityModel` overschrijft de standaard en een modelspecifieke overschrijving voor de bewerking heeft voorrang op beide. Hulpmiddeltaken voeren afzonderlijke modelaanroepen uit en sturen taakspecifieke inhoud naar de geselecteerde modelprovider. Voor het genereren van dashboardtitels worden maximaal de eerste 1.000 tekens van het eerste bericht dat geen opdracht is verzonden; voor vertelling worden het binnenkomende verzoek plus compacte, geredigeerde samenvattingen van hulpmiddelen verzonden. Kies een provider die aansluit bij je vereisten voor kosten en gegevensverwerking.
- `imageModel`: accepteert een tekenreeks (`"provider/model"`) of een object (`{ primary, fallbacks }`).
  - Wordt door het pad van het hulpmiddel `image` gebruikt als configuratie voor het visiemodel wanneer het actieve model geen afbeeldingen kan verwerken. Modellen met ingebouwde visie ontvangen in plaats daarvan de geladen afbeeldingsbytes rechtstreeks.
  - Wordt ook gebruikt als terugvalroutering wanneer het geselecteerde/standaardmodel geen afbeeldingsinvoer kan verwerken.
  - Geef de voorkeur aan expliciete `provider/model`-verwijzingen. Kale ID's worden voor compatibiliteit geaccepteerd; als een kaal ID uniek overeenkomt met een geconfigureerde vermelding met afbeeldingsondersteuning in `models.providers.*.models`, kwalificeert OpenClaw het met die provider. Bij meerdere geconfigureerde overeenkomsten is een expliciet providervoorvoegsel vereist.
- `mediaModels.image`: accepteert een tekenreeks (`"provider/model"`) of een object (`{ primary, fallbacks }`).
  - Wordt gebruikt door de gedeelde mogelijkheid voor het genereren van afbeeldingen en elk toekomstig hulpmiddel- of Plugin-oppervlak dat afbeeldingen genereert.
  - Gebruikelijke waarden: `google/gemini-3.1-flash-image` voor ingebouwde Gemini-afbeeldingsgeneratie, `fal/fal-ai/flux/dev` voor fal, `openai/gpt-image-2` voor OpenAI Images of `openai/gpt-image-1.5` voor OpenAI-uitvoer als PNG/WebP met transparante achtergrond.
  - Als je rechtstreeks een provider/model selecteert, configureer dan ook de bijbehorende providerauthenticatie (bijvoorbeeld `GEMINI_API_KEY` of `GOOGLE_API_KEY` voor `google/*`, `OPENAI_API_KEY` of OpenAI Codex OAuth voor `openai/gpt-image-2` / `openai/gpt-image-1.5`, `FAL_KEY` voor `fal/*`).
  - Als dit wordt weggelaten, kan `image_generate` nog steeds een door authenticatie ondersteunde providerstandaard afleiden. Eerst wordt de huidige standaardprovider geprobeerd, daarna de overige geregistreerde providers voor afbeeldingsgeneratie in volgorde van provider-ID.
- `mediaModels.music`: accepteert een tekenreeks (`"provider/model"`) of een object (`{ primary, fallbacks }`).
  - Wordt gebruikt door de gedeelde mogelijkheid voor het genereren van muziek en het ingebouwde hulpmiddel `music_generate`.
  - Gebruikelijke waarden: `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` of `minimax/music-2.6`.
  - Als dit wordt weggelaten, kan `music_generate` nog steeds een door authenticatie ondersteunde providerstandaard afleiden. Eerst wordt de huidige standaardprovider geprobeerd, daarna de overige geregistreerde providers voor muziekgeneratie in volgorde van provider-ID.
  - Als je rechtstreeks een provider/model selecteert, configureer dan ook de bijbehorende providerauthenticatie/API-sleutel.
- `mediaModels.video`: accepteert een tekenreeks (`"provider/model"`) of een object (`{ primary, fallbacks }`).
  - Wordt gebruikt door de gedeelde mogelijkheid voor het genereren van video's en het ingebouwde hulpmiddel `video_generate`.
  - Gebruikelijke waarden: `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` of `qwen/wan2.7-r2v`.
  - Als dit wordt weggelaten, kan `video_generate` nog steeds een door authenticatie ondersteunde providerstandaard afleiden. Eerst wordt de huidige standaardprovider geprobeerd, daarna de overige geregistreerde providers voor videogeneratie in volgorde van provider-ID.
  - Als je rechtstreeks een provider/model selecteert, configureer dan ook de bijbehorende providerauthenticatie/API-sleutel.
  - De officiële Plugin voor Qwen-videogeneratie ondersteunt maximaal 1 uitvoervideo, 1 invoerafbeelding, 4 invoervideo's, een duur van 10 seconden en de opties `size`, `aspectRatio`, `resolution`, `audio` en `watermark` op providerniveau.
- `pdfModel`: accepteert een tekenreeks (`"provider/model"`) of een object (`{ primary, fallbacks }`).
  - Wordt door het hulpmiddel `pdf` gebruikt voor modelroutering.
  - Als dit wordt weggelaten, valt het PDF-hulpmiddel terug op `imageModel` en vervolgens op het bepaalde sessie-/standaardmodel.
- `pdfMaxMb`: standaardlimiet voor PDF-grootte voor het hulpmiddel `pdf` wanneer `maxBytesMb` niet bij de aanroep wordt doorgegeven.
- `pdfMaxPages`: standaardmaximum voor het aantal pagina's dat door de terugvalmodus voor extractie in het hulpmiddel `pdf` wordt meegenomen.
- `verboseDefault`: standaardniveau voor uitgebreide uitvoer van agents. Waarden: `"off"`, `"on"`, `"full"`. Standaard: `"off"`.
- `toolProgressDetail`: detailmodus voor samenvattingen van het hulpmiddel `/verbose` en hulpmiddelregels in voortgangsconcepten. Waarden: `"explain"` (standaard, compacte menselijk leesbare labels) of `"raw"` (voegt de onbewerkte opdracht/details toe indien beschikbaar). `agents.entries.*.toolProgressDetail` per agent overschrijft deze standaard.
- `reasoningDefault`: standaardzichtbaarheid van redenering voor agents. Waarden: `"off"`, `"on"`, `"stream"`. `agents.entries.*.reasoningDefault` per agent overschrijft deze standaard. Geconfigureerde standaardwaarden voor redenering worden alleen toegepast voor eigenaren, geautoriseerde afzenders of Gateway-contexten voor operatorbeheerders wanneer geen overschrijving van de redenering per bericht of sessie is ingesteld.
- `elevatedDefault`: standaardniveau voor verhoogde uitvoer van agents. Waarden: `"off"`, `"on"`, `"ask"`, `"full"`. Standaard: `"on"`.
- `model.primary`: indeling `provider/model` (bijvoorbeeld `openai/gpt-5.6-sol` voor Codex OAuth-toegang). Als je de provider weglaat, probeert OpenClaw eerst een alias, daarna een unieke overeenkomst met een geconfigureerde provider voor dat exacte model-ID en valt pas daarna terug op de geconfigureerde standaardprovider (verouderd compatibiliteitsgedrag; geef daarom de voorkeur aan expliciete `provider/model`). Als die provider het geconfigureerde standaardmodel niet meer aanbiedt, valt OpenClaw terug op het eerste geconfigureerde provider/model in plaats van een verouderde standaard van een verwijderde provider te tonen.
- `contextTokens`: optionele limiet voor de hele agent. Deze kan het effectieve budget van een groter model verlagen, maar kan een model niet boven de geconfigureerde of gedetecteerde `contextTokens` verhogen. Om één rechtstreeks OpenAI-model het grotere ingebouwde venster te laten gebruiken, stel je `models.providers.openai.models[].contextWindow` en `contextTokens` voor dat model in; zie [standaardwaarden voor het OpenAI-contextvenster](/nl/providers/openai#context-window-defaults-and-long-context-opt-in).
- `models`: geconfigureerde aliassen en instellingen per model. Elke vermelding kan `alias` (snelkoppeling) en `params` (providerspecifiek, bijvoorbeeld `temperature`, `maxTokens`, `cacheRetention`, `context1m`, `responsesServerCompaction`, `responsesCompactThreshold`, OpenRouter-routering via `provider`, `chat_template_kwargs`, `extra_body`/`extraBody`) bevatten. Het toevoegen van vermeldingen beperkt modeloverschrijvingen niet.
  - Gebruik `provider/*`-vermeldingen zoals `"openai/*": {}` of `"vllm/*": {}` om alle gedetecteerde modellen voor geselecteerde providers te tonen zonder elk model-ID handmatig te vermelden.
  - Voeg `agentRuntime` toe aan een `provider/*`-vermelding wanneer elk dynamisch gedetecteerd model voor die provider dezelfde runtime moet gebruiken. Het exacte runtimebeleid van `provider/model` heeft nog steeds voorrang op het jokerteken.
  - Veilige bewerkingen van metagegevens: gebruik `openclaw config set agents.defaults.models '<json>' --strict-json --merge` om vermeldingen toe te voegen. `config set` weigert vervangingen die bestaande vermeldingen zouden verwijderen, tenzij je `--replace` doorgeeft.
- `modelPolicy.allow`: expliciete acceptatielijst voor overschrijvingen. Accepteert aliassen, exacte `provider/model`-verwijzingen en afsluitende jokertekens voor voorvoegsels, zoals `openai/*` of `clawrouter/anthropic/*`. Laat deze weg of gebruik `[]` om elk model toe te staan. `agents.entries.*.modelPolicy.allow` vervangt het standaardbeleid voor die agent; met een expliciete lege lijst staat die agent alle modellen toe.
  - Providergebonden configuratie-/onboardingflows voegen geselecteerde providermodellen samen in deze toewijzing en behouden niet-gerelateerde providers die al zijn geconfigureerd.
  - Voor rechtstreekse OpenAI Responses-modellen wordt serverzijdige Compaction automatisch ingeschakeld. Gebruik `params.responsesServerCompaction: false` om het invoegen van `context_management` te stoppen of `params.responsesCompactThreshold` om de drempel te overschrijven. Zie [serverzijdige Compaction van OpenAI](/nl/providers/openai#advanced-configuration).
- `params`: algemene standaardproviderparameters die op alle modellen worden toegepast. Stel deze in bij `agents.defaults.params` (bijvoorbeeld `{ cacheRetention: "long" }`).
- Samenvoegprioriteit van `params` (configuratie): `agents.defaults.params` (algemene basis) wordt overschreven door `agents.defaults.models["provider/model"].params` (per model), waarna `agents.entries.*.params` (overeenkomend agent-ID) per sleutel overschrijft. Zie [promptcaching](/nl/reference/prompt-caching) voor details.
- `models.providers.openrouter.params.provider`: standaardbeleid voor providerroutering voor heel OpenRouter. OpenClaw stuurt dit door naar het `provider`-object van het OpenRouter-verzoek; `agents.defaults.models["openrouter/<model>"].params.provider` per model en agentparameters overschrijven per sleutel. Zie [OpenRouter-providerroutering](/nl/providers/openrouter#advanced-configuration).
- `params.extra_body`/`params.extraBody`: geavanceerde doorvoer-JSON die wordt samengevoegd in de hoofdtekst van `api: "openai-completions"`-verzoeken voor OpenAI-compatibele proxy's. Als deze botst met gegenereerde verzoeksleutels, heeft de extra hoofdtekst voorrang; niet-native voltooiingsroutes verwijderen daarna nog steeds de uitsluitend voor OpenAI bestemde `store`.
- `params.chat_template_kwargs`: argumenten voor chatsjablonen voor vLLM/OpenAI-compatibele systemen die worden samengevoegd in de hoofdtekst op het hoogste niveau van `api: "openai-completions"`-verzoeken. Voor `vllm/nemotron-3-*` met denken uitgeschakeld verzendt de gebundelde vLLM-Plugin automatisch `enable_thinking: false` en `force_nonempty_content: true`; expliciete `chat_template_kwargs` overschrijven gegenereerde standaardwaarden en `extra_body.chat_template_kwargs` heeft nog steeds de uiteindelijke voorrang. Geconfigureerde denkmodellen van vLLM Qwen en Nemotron bieden binaire `/think`-keuzes (`off`, `on`) in plaats van de inspanningsladder met meerdere niveaus.
- `compat.thinkingFormat`: stijl van de denkpayload voor OpenAI-compatibele systemen. Gebruik `"together"` voor `reasoning.enabled` in Together-stijl, `"qwen"` voor `enable_thinking` op het hoogste niveau in Qwen-stijl, of `"qwen-chat-template"` voor `chat_template_kwargs.enable_thinking` op backends uit de Qwen-familie die chatsjabloon-kwargs op verzoekniveau ondersteunen, zoals vLLM. OpenClaw wijst uitgeschakeld denken toe aan `false` en ingeschakeld denken aan `true`, en geconfigureerde vLLM Qwen-modellen bieden binaire `/think`-keuzes voor deze indelingen.
- `compat.supportedReasoningEfforts`: lijst met OpenAI-compatibele redeneerinspanningen per model. Neem `"xhigh"` op voor aangepaste eindpunten die dit daadwerkelijk accepteren; OpenClaw stelt vervolgens `/think xhigh` beschikbaar in opdrachtmenu's, Gateway-sessierijen, validatie van sessiepatches, validatie van de agent-CLI en validatie van `llm-task` voor die geconfigureerde provider/dat model. Gebruik `compat.reasoningEffortMap` wanneer de backend voor een canoniek niveau een providerspecifieke waarde verwacht.
- `params.preserveThinking`: alleen voor Z.AI beschikbare opt-in voor behouden denkwerk. Wanneer dit is ingeschakeld en denkwerk actief is, verzendt OpenClaw `thinking.clear_thinking: false` en speelt het eerdere `reasoning_content` opnieuw af; zie [Denkwerk en behouden denkwerk van Z.AI](/nl/providers/zai#advanced-configuration).
- `localService`: optionele procesbeheerder op providerniveau voor lokale/zelfgehoste modelservers. Wanneer het geselecteerde model bij die provider hoort, controleert OpenClaw `healthUrl` (of `baseUrl + "/models"`), start het `command` met `args` als het eindpunt niet beschikbaar is, wacht het maximaal `readyTimeoutMs` en verzendt het vervolgens de modelaanvraag. `command` moet een absoluut pad zijn. `idleStopMs: 0` houdt het proces actief totdat OpenClaw wordt afgesloten; een positieve waarde stopt het door OpenClaw gestarte proces na dat aantal milliseconden inactiviteit. Zie [Lokale modelservices](/nl/gateway/local-model-services).
- Runtimebeleid hoort bij providers of modellen, niet bij `agents.defaults`. Gebruik `models.providers.<provider>.agentRuntime` voor providerbrede regels of `agents.defaults.models["provider/model"].agentRuntime` / `agents.entries.*.models["provider/model"].agentRuntime` voor modelspecifieke regels. Alleen een provider-/modelvoorvoegsel selecteert nooit een harness. Als de runtime niet is ingesteld of `auto` is, mag OpenAI Codex alleen impliciet selecteren voor een exacte officiële HTTPS-route voor Platform Responses of ChatGPT Responses zonder een handmatig ingestelde aanvraagoverschrijving. Zie [Impliciete OpenAI-agentruntime](/nl/providers/openai#implicit-agent-runtime).
- Configuratieschrijvers die deze velden wijzigen (bijvoorbeeld `/models set`, `/models set-image` en opdrachten om fallbacks toe te voegen of te verwijderen) slaan de canonieke objectvorm op en behouden waar mogelijk bestaande fallbacklijsten.
- `maxConcurrent`: maximaal aantal parallelle agentuitvoeringen in verschillende sessies (elke sessie wordt nog steeds serieel verwerkt). Standaard: `4`.

### Runtimebeleid

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: { id: "codex" },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
        "vllm/*": {
          agentRuntime: { id: "openclaw" },
        },
      },
    },
  },
}
```

- `id`: `"auto"`, `"openclaw"`, een geregistreerde plugin-harness-id of een ondersteunde CLI-backendalias. De gebundelde Codex-plugin registreert `codex`; de gebundelde Anthropic-plugin biedt de CLI-backend `claude-cli`.
- `id: "auto"` laat geregistreerde plugin-harnassen effectieve routes claimen die hun ondersteuningscontract declareren of daar anderszins aan voldoen, en gebruikt OpenClaw wanneer geen harness overeenkomt. Een expliciete pluginruntime zoals `id: "codex"` vereist die harness en een compatibele effectieve route; deze faalt gesloten als een van beide niet beschikbaar is of als de uitvoering mislukt.
- `id: "pi"` wordt alleen geaccepteerd als verouderde alias voor `openclaw` om uitgebrachte configuraties van v2026.5.22 en eerder te behouden. Nieuwe configuraties moeten `openclaw` gebruiken.
- De runtimeprioriteit is eerst het exacte modelbeleid (`agents.entries.*.models["provider/model"]`, `agents.defaults.models["provider/model"]` of `models.providers.<provider>.models[]`), daarna `agents.entries.*` / `agents.defaults.models["provider/*"]` en vervolgens het providerbrede beleid bij `models.providers.<provider>.agentRuntime`.
- Runtime-sleutels voor de volledige agent zijn verouderd. `agents.defaults.agentRuntime`, `agents.entries.*.agentRuntime`, runtime-pins voor sessies en `OPENCLAW_AGENT_RUNTIME` worden genegeerd bij de runtimeselectie. Voer `openclaw doctor --fix` uit om verouderde waarden te verwijderen.
- Geschikte exacte officiële HTTPS-routes voor OpenAI Responses/ChatGPT zonder handmatig ingestelde aanvraagoverschrijving mogen de Codex-harness impliciet gebruiken. Provider/model `agentRuntime.id: "codex"` maakt Codex een fail-closed-vereiste, maar maakt een incompatibele route niet compatibel.
- Geef voor Claude CLI-implementaties de voorkeur aan `model: "anthropic/claude-opus-5"` plus modelgebonden `agentRuntime.id: "claude-cli"`. Verouderde `claude-cli/<model>`-referenties blijven werken voor compatibiliteit, maar nieuwe configuraties moeten de provider-/modelselectie canoniek houden en de uitvoeringsbackend in het runtimebeleid voor de provider/het model plaatsen.
- Dit beheert alleen de uitvoering van agentbeurten met tekst. Mediageneratie, beeldverwerking, PDF, muziek, video en TTS blijven hun provider-/modelinstellingen gebruiken.

**Ingebouwde verkorte aliassen** (alleen van toepassing wanneer het model in `agents.defaults.models` staat):

| Alias               | Model                           |
| ------------------- | ------------------------------- |
| `opus`              | `anthropic/claude-opus-5`       |
| `sonnet`            | `anthropic/claude-sonnet-5`     |
| `gpt`               | `openai/gpt-5.4`                |
| `gpt-mini`          | `openai/gpt-5.4-mini`           |
| `gpt-nano`          | `openai/gpt-5.4-nano`           |
| `gemini`            | `google/gemini-3.1-pro-preview` |
| `gemini-flash`      | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite`  |

Jouw geconfigureerde aliassen hebben altijd voorrang op de standaardwaarden.

Z.AI GLM-4.x-modellen schakelen automatisch de denkmodus in, tenzij je `--thinking off` instelt of zelf `agents.defaults.models["zai/<model>"].params.thinking` definieert.
Z.AI-modellen schakelen `tool_stream` standaard in voor het streamen van toolaanroepen. Stel `agents.defaults.models["zai/<model>"].params.tool_stream` in op `false` om dit uit te schakelen.
Bij Anthropic Claude Opus 4.8 blijft denken standaard uitgeschakeld in OpenClaw; wanneer adaptief denken expliciet wordt ingeschakeld, is de door Anthropic beheerde standaardwaarde voor inspanning `high`. Claude 4.6-modellen gebruiken standaard `adaptive` wanneer geen expliciet denkniveau is ingesteld.

### CLI-backendselectie

De werking van CLI-adapters wordt door plugins geregistreerd en niet onder de standaardwaarden
van agents geconfigureerd. Selecteer een geregistreerde CLI-backend met modelgebonden `agentRuntime.id`,
zoals hierboven weergegeven. Zie [CLI-backends](/nl/gateway/cli-backends) voor de werking en
[CLI-backendplugins bouwen](/nl/plugins/cli-backend-plugins) voor de registratie van opdrachten,
sessies, afbeeldingen en parsers.

### `agents.defaults.promptOverlays`

Provideronafhankelijke promptoverlays die per modelfamilie worden toegepast op door OpenClaw samengestelde promptoppervlakken. Model-id's uit de GPT-5-familie ontvangen het gedeelde gedragscontract voor OpenClaw-/providerroutes; `personality` beheert alleen de vriendelijke interactiestijllaag. Native Codex-app-serverroutes behouden door Codex beheerde basis-/modelinstructies in plaats van deze OpenClaw GPT-5-overlay, en OpenClaw schakelt de ingebouwde persoonlijkheid van Codex uit voor native threads.

```json5
{
  agents: {
    defaults: {
      promptOverlays: {
        gpt5: {
          personality: "friendly", // vriendelijk | aan | uit
        },
      },
    },
  },
}
```

- `"friendly"` (standaard) en `"on"` schakelen de vriendelijke interactiestijllaag in.
- `"off"` schakelt alleen de vriendelijke laag uit; het getagde GPT-5-gedragscontract blijft ingeschakeld.
- Het verouderde `plugins.entries.openai.config.personality` wordt nog steeds gelezen wanneer deze gedeelde instelling niet is ingesteld.

### `agents.defaults.heartbeat`

Periodieke Heartbeat-uitvoeringen.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m schakelt uit
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // standaard: true; false laat de Heartbeat-sectie weg uit de systeemprompt
        lightContext: false, // standaard: false; true slaat workspace-bootstrapbestanden over voor Heartbeat-uitvoeringen
        isolatedSession: false, // standaard: false; true voert elke Heartbeat uit in een nieuwe sessie (zonder gespreksgeschiedenis)
        skipWhenBusy: false, // standaard: false; true wacht ook op de subagent-/geneste lanes van deze agent
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (standaard) | block
        target: "none", // standaard: none | opties: last | whatsapp | telegram | discord | ...
        prompt: "Volg de tijdelijke context van de Heartbeat-monitor...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
        timeoutSeconds: 45,
      },
    },
  },
}
```

- `every`: duurtekenreeks (ms/s/m/h). Standaard: `30m` (API-sleutelauthenticatie) of `1h` (OAuth-authenticatie). Stel in op `0m` om uit te schakelen.
- Het interval wordt naar een systeemeigen Cron-monitorrij geschreven. Voer `openclaw doctor --fix` uit om een ontbrekende of verouderde rij aan te maken. Als Cron is uitgeschakeld, worden geplande Heartbeats niet uitgevoerd en registreert de Gateway bij het opstarten een waarschuwing.
- `includeSystemPromptSection`: wanneer false, wordt de Heartbeat-sectie weggelaten uit de systeemprompt. Standaard: `true`.
- `suppressToolErrorWarnings`: wanneer true, worden waarschuwingspayloads voor toolfouten tijdens Heartbeat-uitvoeringen onderdrukt.
- `timeoutSeconds`: de maximale toegestane tijd in seconden voor een Heartbeat-agentbeurt voordat deze wordt afgebroken. Laat dit oningesteld om `agents.defaults.timeoutSeconds` te gebruiken wanneer dat is ingesteld; anders wordt het Heartbeat-interval gebruikt, begrensd op 600 seconden.
- `directPolicy`: beleid voor directe/DM-bezorging. `allow` (standaard) staat bezorging aan directe doelen toe. `block` onderdrukt bezorging aan directe doelen en genereert `reason=dm-blocked`.
- `lightContext`: wanneer true, gebruiken Heartbeat-uitvoeringen een lichtgewicht bootstrapcontext en slaan ze workspace-bootstrapbestanden over. De tijdelijke monitorcontext wordt in beide gevallen door de Heartbeat-runner geïnjecteerd.
- `isolatedSession`: wanneer true, wordt elke Heartbeat uitgevoerd in een nieuwe sessie zonder eerdere gespreksgeschiedenis. Hetzelfde isolatiepatroon als Cron `sessionTarget: "isolated"`. Vermindert de tokenkosten per Heartbeat van ~100K tot ~2-5K tokens.
- `skipWhenBusy`: wanneer true, worden Heartbeat-uitvoeringen uitgesteld vanwege extra bezette lanes van die agent: het eigen sessiesleutelgebonden subagentwerk of geneste opdrachtwerk. Cron-lanes stellen Heartbeats altijd uit, ook zonder deze vlag.
- Per agent: stel `agents.entries.*.heartbeat` in. Wanneer een agent `heartbeat` definieert, voeren **alleen die agents** Heartbeats uit.
- Heartbeats voeren volledige agentbeurten uit — kortere intervallen verbruiken meer tokens.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // standaard | beveiliging
        provider: "my-provider", // id van een geregistreerde Compaction-providerplugin (optioneel)
        thinkingLevel: "low", // optionele denkoverschrijving uitsluitend voor Compaction
        timeoutSeconds: 180,
        keepRecentTokens: 50000,
        recentTurnsPreserve: 3,
        identifierPolicy: "strict", // strikt | uit
        qualityGuard: { enabled: true, maxRetries: 1 },
        midTurnPrecheck: { enabled: false }, // optionele controle van de tool-loopdruk
        postIndexSync: "async", // uit | asynchroon | wachten
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-6", // optionele modeloverschrijving uitsluitend voor Compaction
        truncateAfterCompaction: true, // na Compaction roteren naar een kleinere opvolgende JSONL
        maxActiveTranscriptBytes: "20mb", // optionele lokale Compaction-trigger tijdens de voorafgaande controle
        notifyUser: true, // meldingen wanneer Compaction begint/voltooid is en bij degradatie van het geheugen doorspoelen (standaard: false)
        memoryFlush: {
          enabled: true,
          model: "ollama/qwen3:8b", // optionele modeloverschrijving uitsluitend voor het geheugen doorspoelen
          softThresholdTokens: 6000,
          forceFlushTranscriptBytes: "2mb",
        },
      },
    },
  },
}
```

- `mode`: `default` of `safeguard` (samenvatting in delen voor lange geschiedenissen). Zie [Compaction](/nl/concepts/compaction).
- `provider`: id van een geregistreerde Compaction-providerplugin. Wanneer dit is ingesteld, wordt de `summarize()` van de provider aangeroepen in plaats van de ingebouwde LLM-samenvatting. Valt bij een fout terug op de ingebouwde functie. Door een provider in te stellen, wordt `mode: "safeguard"` afgedwongen. Zie [Compaction](/nl/concepts/compaction).
- `thinkingLevel`: optioneel denkniveau dat alleen wordt gebruikt voor ingebedde OpenClaw-Compaction-samenvattingen (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max` of `ultra`). Dit overschrijft het huidige denkniveau van de sessie en wordt begrensd op basis van het geselecteerde Compaction-model/de geselecteerde runtime. Laat dit oningesteld om het sessieniveau over te nemen. Compaction via de systeemeigen Codex-appserver negeert deze instelling, omdat het systeemeigen compact-verzoek geen denkniveau per bewerking kan overschrijven; OpenClaw registreert een waarschuwing wanneer dit is geconfigureerd.
- `timeoutSeconds`: maximaal aantal seconden dat één Compaction-bewerking mag duren voordat OpenClaw deze afbreekt. Standaard: `180`.
- `keepRecentTokens`: budget voor het afkappunt van de agent om het meest recente uiteinde van het transcript woordelijk te behouden. Handmatige `/compact` respecteert dit wanneer het expliciet is ingesteld; anders is handmatige Compaction een hard controlepunt.
- `recentTurnsPreserve`: aantal meest recente beurten van gebruiker/assistent dat buiten de beveiligingssamenvatting woordelijk wordt behouden. Standaard: `3`.
- `identifierPolicy`: `strict` (standaard) of `off`. `strict` voegt tijdens de Compaction-samenvatting ingebouwde richtlijnen voor het behoud van ondoorzichtige identificatoren vooraan toe.
- `qualityGuard`: controles waarbij bij onjuist gevormde uitvoer opnieuw wordt geprobeerd voor beveiligingssamenvattingen. Standaard ingeschakeld in de beveiligingsmodus; stel `enabled: false` in om de controle over te slaan.
- `midTurnPrecheck`: optionele controle op druk in de tool-lus. Wanneer `enabled: true`, controleert OpenClaw de contextdruk nadat toolresultaten zijn toegevoegd en vóór de volgende modelaanroep. Als de context niet meer past, wordt de huidige poging afgebroken voordat de prompt wordt verzonden en wordt het bestaande herstelpad van de voorafgaande controle hergebruikt om toolresultaten af te kappen of Compaction uit te voeren en het opnieuw te proberen. Werkt met zowel de Compaction-modus `default` als `safeguard`. Standaard: uitgeschakeld.
- `postIndexSync`: herindexeringsmodus voor het sessiegeheugen na Compaction. Standaard: `"async"`. Gebruik `"await"` voor de grootste actualiteit, `"async"` voor een lagere Compaction-latentie of `"off"` alleen wanneer de synchronisatie van het sessiegeheugen elders wordt afgehandeld.
- `postCompactionSections`: optionele namen van H2/H3-secties in AGENTS.md die na Compaction opnieuw moeten worden geïnjecteerd. Laat dit oningesteld of gebruik `[]` om dit uit te schakelen.
- `model`: optionele `provider/model-id` of kale alias uit `agents.defaults.models`, uitsluitend voor Compaction-samenvattingen. Kale aliassen worden vóór verzending omgezet; geconfigureerde letterlijke model-id's behouden voorrang bij conflicten. Gebruik dit wanneer de hoofdsessie één model moet blijven gebruiken, maar Compaction-samenvattingen op een ander model moeten worden uitgevoerd; wanneer dit niet is ingesteld, gebruikt Compaction het primaire model van de sessie.
- `truncateAfterCompaction`: roteert het actieve sessietranscript na Compaction, zodat toekomstige beurten alleen de samenvatting en het niet-samengevatte uiteinde laden, terwijl het vorige volledige transcript gearchiveerd blijft. Voorkomt onbeperkte groei van het actieve transcript in langdurige sessies. Standaard: `false`.
- `maxActiveTranscriptBytes`: optionele drempelwaarde in bytes (`number` of tekenreeksen zoals `"20mb"`) die vóór een uitvoering normale lokale Compaction activeert wanneer de transcriptgeschiedenis de drempel overschrijdt. Vereist `truncateAfterCompaction`, zodat een geslaagde Compaction naar een kleiner opvolgend transcript kan roteren. Uitgeschakeld wanneer dit niet is ingesteld of `0`.
- `notifyUser`: wanneer `true`, worden korte meldingen over contextonderhoud naar de gebruiker gestuurd: wanneer Compaction begint en is voltooid (bijvoorbeeld: "Context wordt gecomprimeerd..." en "Compaction voltooid"), en wanneer een geheugenopslag vóór Compaction is uitgeput, zodat het antwoord in een beperkte toestand doorgaat (bijvoorbeeld: "Geheugenonderhoud is tijdelijk mislukt; je antwoord wordt voortgezet."). Standaard uitgeschakeld om deze meldingen stil te houden.
- `memoryFlush`: stille agentische beurt vóór automatische Compaction om duurzame herinneringen op te slaan. Stel `model` in op een exacte provider/een exact model, zoals `ollama/qwen3:8b`, wanneer deze onderhoudsbeurt op een lokaal model moet blijven; de overschrijving neemt de actieve terugvalketen van de sessie niet over. `forceFlushTranscriptBytes` dwingt de opslag af wanneer de transcriptgrootte de drempel bereikt, zelfs als de tokentellers verouderd zijn. Wordt overgeslagen wanneer de werkruimte alleen-lezen is.

Aangepaste Compaction-instructies worden door de code beheerd. Implementeer een Compaction-provider-
plugin met `summarize()` voor aangepaste samenstellingslogica van samenvattingen en gebruik
`before_prompt_build` wanneer context na Compaction in latere
modelprompts moet worden geïnjecteerd. Doctor verwijdert de buiten gebruik gestelde instructievelden en verwijst naar deze
aansluitpunten.

### `agents.defaults.contextPruning`

Snoeit **oude toolresultaten** uit de context in het geheugen voordat deze naar het LLM wordt verzonden. Wijzigt de sessiegeschiedenis op schijf **niet**. Standaard uitgeschakeld; stel `mode: "cache-ttl"` in om dit in te schakelen.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // uit (standaard) | cache-ttl
      },
    },
  },
}
```

<Accordion title="Gedrag van de cache-ttl-modus">

- `mode: "cache-ttl"` schakelt snoeirondes in.
- Bij het snoeien worden te grote toolresultaten eerst licht ingekort en worden oudere toolresultaten daarna, indien nodig, volledig gewist.

**Licht inkorten** behoudt het begin en einde en voegt `...` in het midden in.

**Volledig wissen** vervangt het volledige toolresultaat door de tijdelijke aanduiding.

Opmerkingen:

- Afbeeldingsblokken worden nooit ingekort/gewist.
- Verhoudingen zijn gebaseerd op tekens (bij benadering), niet op exacte aantallen tokens.
- De meest recente assistentberichten blijven behouden.

</Accordion>

Zie [Sessies snoeien](/nl/concepts/session-pruning) voor details over het gedrag.

### Blokstreaming

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200, breakPreference: "paragraph" },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off (standaard) | natural | custom (gebruik minMs/maxMs)
    },
  },
}
```

- Voor andere kanalen dan Telegram is expliciete `*.streaming.block.enabled: true` vereist om blokantwoorden in te schakelen. QQ Bot is de uitzondering: het heeft geen `streaming.block`-sleutels en streamt blokantwoorden tenzij `channels.qqbot.streaming.mode` `"off"` is.
- Overschrijvingen per kanaal: `channels.<channel>.streaming.block.coalesce` (en varianten per account). Discord, Google Chat, Mattermost, MS Teams, Signal en Slack gebruiken standaard `minChars: 1500` / `idleMs: 1000`.
- `blockStreamingChunk.breakPreference`: gewenste blokgrens (`"paragraph" | "newline" | "sentence"`).
- `humanDelay`: willekeurige pauze tussen blokantwoorden. Standaard: `off`. `natural` = 800-2500ms. `custom` gebruikt `minMs`/`maxMs` (valt voor elke niet-ingestelde grens terug op het natuurlijke bereik). Overschrijving per agent: `agents.entries.*.humanDelay`.

Zie [Streaming](/nl/concepts/streaming) voor details over gedrag en opdeling.

### Typindicatoren

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- Standaardwaarden: `instant` voor directe chats/vermeldingen, `message` voor groepschats zonder vermelding.
- Standaard voor `typingIntervalSeconds`: `6`.
- Overschrijving per agent: `agents.entries.*.typingMode`.

Zie [Typindicatoren](/nl/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Optionele sandboxing voor de ingebedde agent. Zie [Sandboxing](/nl/gateway/sandboxing) voor de volledige handleiding.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off (standaard) | non-main | all
        backend: "docker", // docker (standaard) | ssh | openshell
        scope: "agent", // session | agent (standaard) | shared
        workspaceAccess: "none", // none (standaard) | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          gpus: "all",
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs / inline-inhoud wordt ook ondersteund:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

De hierboven weergegeven standaardwaarden (`off`/`docker`/`agent`/`none`/`bookworm-slim`-image/`none`-netwerk/enz.) zijn de daadwerkelijke standaardwaarden van OpenClaw, niet slechts illustratieve waarden.

<Accordion title="Sandboxdetails">

**Backend:**

- `docker`: lokale Docker-runtime (standaard)
- `ssh`: algemene externe runtime op basis van SSH
- `openshell`: OpenShell-runtime

Wanneer `backend: "openshell"` is geselecteerd, worden runtimespecifieke instellingen verplaatst naar
`plugins.entries.openshell.config`.

**Configuratie van de SSH-backend:**

- `target`: SSH-doel in `user@host[:port]`-vorm
- `command`: SSH-clientopdracht (standaard: `ssh`)
- `workspaceRoot`: absolute externe hoofdmap die wordt gebruikt voor werkruimten per bereik (standaard: `/tmp/openclaw-sandboxes`)
- `identityFile` / `certificateFile` / `knownHostsFile`: bestaande lokale bestanden die aan OpenSSH worden doorgegeven
- `identityData` / `certificateData` / `knownHostsData`: inline-inhoud of SecretRefs die OpenClaw tijdens runtime omzet in tijdelijke bestanden
- `strictHostKeyChecking` / `updateHostKeys`: OpenSSH-instellingen voor host-sleutelbeleid (beide standaard `true`)

**Voorrangsvolgorde voor SSH-authenticatie:**

- `identityData` heeft voorrang op `identityFile`
- `certificateData` heeft voorrang op `certificateFile`
- `knownHostsData` heeft voorrang op `knownHostsFile`
- Door SecretRef ondersteunde `*Data`-waarden worden opgelost vanuit de actieve momentopname van de secrets-runtime voordat de sandboxsessie start

**Gedrag van de SSH-backend:**

- vult de externe werkruimte eenmaal na aanmaken of opnieuw aanmaken
- houdt daarna de externe SSH-werkruimte canoniek
- leidt `exec`, bestandstools en mediapaden via SSH
- synchroniseert externe wijzigingen niet automatisch terug naar de host
- ondersteunt geen sandbox-browsercontainers

**Toegang tot de werkruimte:**

- `none`: sandboxwerkruimte per bereik onder `~/.openclaw/sandboxes` (standaard)
- `ro`: sandboxwerkruimte op `/workspace`, agentwerkruimte alleen-lezen gekoppeld op `/agent`
- `rw`: agentwerkruimte leesbaar/schrijfbaar gekoppeld op `/workspace`

**Bereik:**

- `session`: container + werkruimte per sessie
- `agent`: één container + werkruimte per agent (standaard)
- `shared`: gedeelde container en werkruimte (geen isolatie tussen sessies)

**OpenShell-pluginconfiguratie:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // spiegel (standaard) | extern
          command: "openshell",
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // optioneel
          gatewayEndpoint: "https://lab.example", // optioneel
          policy: "strict", // optionele OpenShell-beleids-id
          providers: ["openai"], // optioneel
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**OpenShell-modus:**

- `mirror`: vul extern vanuit lokaal vóór uitvoering, synchroniseer terug na uitvoering; de lokale werkruimte blijft canoniek
- `remote`: vul extern eenmaal wanneer de sandbox wordt aangemaakt en houd daarna de externe werkruimte canoniek

In de modus `remote` worden lokale hostbewerkingen die buiten OpenClaw zijn gemaakt na de vulstap niet automatisch naar de sandbox gesynchroniseerd.
Het transport verloopt via SSH naar de OpenShell-sandbox, maar de plugin beheert de levenscyclus van de sandbox en de optionele spiegelsynchronisatie.

**`setupCommand`** wordt eenmaal uitgevoerd nadat de container is aangemaakt (via `sh -lc`). Vereist uitgaand netwerkverkeer, een beschrijfbare hoofdmap en de rootgebruiker.

**Containers gebruiken standaard `network: "none"`** — stel dit in op `"bridge"` (of een aangepast bridgenetwerk) als de agent uitgaande toegang nodig heeft.
`"host"` wordt geblokkeerd. `"container:<id>"` wordt standaard geblokkeerd, tenzij je expliciet
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` instelt (noodmaatregel).
Codex-app-serverbeurten in een actieve OpenClaw-sandbox gebruiken dezelfde instelling voor uitgaand verkeer voor hun systeemeigen netwerktoegang in codemodus.

**Binnenkomende bijlagen** worden klaargezet in `media/inbound/*` in de actieve werkruimte.

**`docker.binds`** koppelt aanvullende hostmappen; globale en agentspecifieke koppelingen worden samengevoegd.

**Sandboxbrowser** (`sandbox.browser.enabled`, standaard `false`): Chromium + CDP in een container. noVNC-URL wordt in de systeemprompt geïnjecteerd. Vereist geen `browser.enabled` in `openclaw.json`.
Waarnemerstoegang via noVNC gebruikt standaard VNC-authenticatie en OpenClaw genereert een URL met een kortlevend token (in plaats van het wachtwoord in de gedeelde URL bloot te stellen).

- `allowHostControl: false` (standaard) voorkomt dat sandboxsessies de hostbrowser als doel gebruiken.
- `network` is standaard `openclaw-sandbox-browser` (speciaal bridgenetwerk). Stel dit alleen in op `bridge` als je expliciet algemene bridgeconnectiviteit wilt. `"host"` wordt hier ook geblokkeerd.
- `cdpSourceRange` beperkt CDP-toegang bij de containerrand optioneel tot een CIDR-bereik (bijvoorbeeld `172.21.0.1/32`).
- `sandbox.browser.binds` koppelt aanvullende hostmappen uitsluitend in de sandbox-browsercontainer. Wanneer dit is ingesteld (inclusief `[]`), vervangt het `docker.binds` voor de browsercontainer.
- Chromium in de sandbox-browsercontainer wordt altijd gestart met `--no-sandbox --disable-setuid-sandbox` (containers beschikken niet over de kernelprimitieven die Chromes eigen sandbox nodig heeft); hiervoor bestaat geen configuratieschakelaar.
- De standaardinstellingen voor het starten zijn gedefinieerd in `scripts/sandbox-browser-entrypoint.sh` en afgestemd op containerhosts:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--password-store=basic`
  - `--use-mock-keychain`
  - `--disable-3d-apis`, `--disable-gpu` en `--disable-software-rasterizer` zijn
    standaard ingeschakeld en kunnen worden uitgeschakeld met
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` als dit vereist is voor het gebruik van WebGL/3D.
  - `--disable-extensions` (standaard ingeschakeld); `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0`
    schakelt extensies opnieuw in als je workflow ervan afhankelijk is.
  - `--renderer-process-limit=2` standaard; wijzig dit met
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`, stel `0` in om de
    standaardproceslimiet van Chromium te gebruiken.
  - `--headless=new` alleen wanneer `headless` is ingeschakeld.
  - De standaardwaarden zijn de basisinstellingen van de containerimage; gebruik een aangepaste browserimage met een aangepast
    toegangspunt om de standaardwaarden van de container te wijzigen.

</Accordion>

Browsersandboxing en `sandbox.docker.binds` werken alleen met Docker.

Images bouwen (vanuit een broncodecheckout):

```bash
scripts/sandbox-setup.sh           # hoofdimage voor de sandbox
scripts/sandbox-browser-setup.sh   # optionele browserimage
```

Zie voor npm-installaties zonder broncodecheckout [Sandboxing § Images en installatie](/nl/gateway/sandboxing#images-and-setup) voor inline `docker build`-opdrachten.

### `agents.entries` (overschrijvingen per agent)

Gebruik `agents.entries.*.tts` om een agent een eigen TTS-provider, stem, model,
stijl of automatische TTS-modus te geven. Het agentblok wordt diep samengevoegd met de globale
`tts`, zodat gedeelde aanmeldgegevens op één plaats kunnen blijven terwijl afzonderlijke
agents alleen de benodigde stem- of providervelden overschrijven. De overschrijving van de actieve agent
is van toepassing op automatische gesproken antwoorden, `/tts audio`, `/tts status` en
de agenttool `tts`. Zie [Tekst-naar-spraak](/nl/tools/tts#per-agent-voice-overrides)
voor providervoorbeelden en de voorrangsvolgorde.

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // of { primary, fallbacks }
        utilityModel: "openai/gpt-5.4-mini",
        thinkingDefault: "high", // overschrijving van het denkniveau per agent
        reasoningDefault: "on", // overschrijving van de zichtbaarheid van redeneringen per agent
        fastModeDefault: false, // overschrijving van de snelle modus per agent
        params: { cacheRetention: "none" }, // overschrijft overeenkomende defaults.models-parameters per sleutel
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
        skills: ["docs-search"], // vervangt agents.defaults.skills wanneer ingesteld
        identity: {
          name: "Samantha",
          theme: "behulpzame luiaard",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent", // persistent | oneshot
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: stabiele agent-id (verplicht).
- `default`: wanneer er meerdere zijn ingesteld, wint de eerste (waarschuwing wordt gelogd). Als er geen is ingesteld, is het eerste item in de lijst de standaardwaarde.
- `model`: de tekenreeksvorm stelt een strikt primair model per agent in zonder modelfallback; de objectvorm `{ primary }` is ook strikt, tenzij je `fallbacks` toevoegt. Gebruik `{ primary, fallbacks: [...] }` om fallback voor die agent in te schakelen, of `{ primary, fallbacks: [] }` om strikt gedrag expliciet te maken. Cron-taken die alleen `primary` overschrijven, nemen nog steeds de standaardfallbacks over, tenzij je `fallbacks: []` instelt.
- `utilityModel`: optionele overschrijving per agent voor korte interne taken, zoals gegenereerde sessie- en threadtitels. Valt terug op `agents.defaults.utilityModel` en vervolgens op de gedeclareerde standaardwaarde voor kleine modellen van de effectieve sessieprovider. Dashboardtitels proberen het eenmaal opnieuw met het effectieve reguliere sessiemodel. Een lege tekenreeks slaat de alternatieve hulproute voor deze agent over zonder het genereren van dashboardtitels uit te schakelen.
- `params`: streamparameters per agent die worden samengevoegd over de geselecteerde modelvermelding in `agents.defaults.models`. Gebruik dit voor agentspecifieke overschrijvingen zoals `cacheRetention`, `temperature` of `maxTokens`, zonder de volledige modelcatalogus te dupliceren.
- `tts`: optionele tekst-naar-spraakoverschrijvingen per agent. Het blok wordt diep samengevoegd over `tts`, dus bewaar gedeelde providerreferenties en het fallbackbeleid in `tts` en stel hier alleen personaspecifieke waarden in, zoals provider, stem, model, stijl of automatische modus.
- `skills`: optionele allowlist voor Skills per agent. Indien weggelaten, neemt de agent `agents.defaults.skills` over wanneer dit is ingesteld; een expliciete lijst vervangt de standaardwaarden in plaats van ermee te worden samengevoegd, en `[]` betekent geen Skills.
- `thinkingDefault`: optioneel standaarddenkniveau per agent (`off | minimal | low | medium | high | xhigh | adaptive | max`). Overschrijft `agents.defaults.thinkingDefault` voor deze agent wanneer er geen overschrijving per bericht of sessie is ingesteld. Het geselecteerde provider-/modelprofiel bepaalt welke waarden geldig zijn; voor Google Gemini behoudt `adaptive` het door de provider beheerde dynamische denken (`thinkingLevel` weggelaten bij Gemini 3/3.1, `thinkingBudget: -1` bij Gemini 2.5).
- `reasoningDefault`: optionele standaardzichtbaarheid van redeneringen per agent (`on | off | stream`). Overschrijft `agents.defaults.reasoningDefault` voor deze agent wanneer er geen overschrijving van redeneringen per bericht of sessie is ingesteld.
- `fastModeDefault`: optionele standaardwaarde per agent voor de snelle modus (`"auto" | true | false`). Wordt toegepast wanneer er geen overschrijving van de snelle modus per bericht of sessie is ingesteld.
- `models`: optionele overschrijvingen per agent voor de modelcatalogus/runtime, geïndexeerd op volledige `provider/model`-id's. Gebruik `models["provider/model"].agentRuntime` voor runtime-uitzonderingen per agent.
- `runtime`: optionele runtimebeschrijving per agent. Gebruik `type: "acp"` met de standaardwaarden van `runtime.acp` (`agent`, `backend`, `mode`, `cwd`) wanneer de agent standaard ACP-harnesssessies moet gebruiken.
- `identity.avatar`: werkruimte-relatief pad, `http(s)`-URL of `data:`-URI.
- Lokale werkruimte-relatieve `identity.avatar`-afbeeldingsbestanden zijn beperkt tot 2 MB. `http(s)`-URL's en `data:`-URI's worden niet getoetst aan de lokale bestandsgroottelimiet.
- `identity` leidt standaardwaarden af: `ackReaction` uit `emoji`, `mentionPatterns` uit `name`/`emoji`.
- `subagents.allowAgents`: allowlist van geconfigureerde agent-id's voor expliciete `sessions_spawn.agentId`-doelen (`["*"]` = elk geconfigureerd doel; standaard: alleen dezelfde agent). Neem de id van de aanvrager op wanneer op zichzelf gerichte `agentId`-aanroepen toegestaan moeten zijn. Verouderde vermeldingen waarvan de agentconfiguratie is verwijderd, worden door `sessions_spawn` geweigerd en uit `agents_list` weggelaten; voer `openclaw doctor --fix` uit om ze op te ruimen, of voeg een minimale `agents.entries.*`-vermelding toe als dat doel startbaar moet blijven en tegelijk de standaardwaarden moet overnemen.
- Beveiliging voor sandbox-overerving: als de sessie van de aanvrager in een sandbox draait, weigert `sessions_spawn` doelen die zonder sandbox zouden worden uitgevoerd.
- `subagents.requireAgentId`: blokkeer wanneer dit waar is `sessions_spawn`-aanroepen die `agentId` weglaten (dwingt expliciete profielselectie af; standaard: false).
- `subagents.maxConcurrent`: maximaal aantal gelijktijdige uitvoeringen van onderliggende agents voor alle subagentuitvoeringen. Standaard: `8`.
- `subagents.maxChildrenPerAgent`: maximaal aantal actieve onderliggende agents dat één agentsessie kan starten. Standaard: `5`.
- `subagents.maxSpawnDepth`: maximale nestingsdiepte voor het starten van subagents (`1`-`5`). Standaard: `1` (geen nesting).
- `subagents.archiveAfterMinutes`: tijd waarna de status van een voltooide subagent wordt gearchiveerd. Standaard: `60`.

---

## Routering met meerdere agents

Voer meerdere geïsoleerde agents uit binnen één Gateway. Zie [Meerdere agents](/nl/concepts/multi-agent).

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### Overeenkomstvelden voor bindingen

- `type` (optioneel): `route` voor normale routering (een ontbrekend type gebruikt standaard route), `acp` voor persistente ACP-gespreksbindingen.
- `match.channel` (verplicht)
- `match.accountId` (optioneel; `*` = elk account; weggelaten = standaardaccount)
- `match.peer` (optioneel; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (optioneel; kanaalspecifiek)
- `acp` (optioneel; alleen voor `type: "acp"`): `{ mode, label, cwd, backend }`

**Deterministische overeenkomstvolgorde:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (exact, zonder peer/guild/team)
5. `match.accountId: "*"` (kanaalbreed)
6. Standaardagent

Binnen elk niveau wint de eerste overeenkomende `bindings`-vermelding.

Voor `type: "acp"`-vermeldingen zoekt OpenClaw op exacte gespreksidentiteit (`match.channel` + account + `match.peer.id`) en gebruikt het niet de bovenstaande niveauvolgorde voor routebindingen.

### Toegangsprofielen per agent

<Accordion title="Volledige toegang (geen sandbox)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Alleen-lezen-tools + werkruimte">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Geen toegang tot het bestandssysteem (alleen berichten)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

Zie [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) voor details over de prioriteitsvolgorde.

---

## Sessie

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 30 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    maintenance: {
      mode: "enforce", // enforce (standaard) | warn
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // duur of false
      maxDiskBytes: "500mb", // optioneel hard budget
      highWaterBytes: "400mb", // optioneel opschoondoel
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // standaard automatisch opheffen van focus na inactiviteit in uren (`0` schakelt dit uit)
      maxAgeHours: 0, // standaard absolute maximumleeftijd in uren (`0` schakelt dit uit)
    },
    sharing: {
      readOnly: true,
      suggest: true,
      drafts: true,
    },
    mainKey: "main", // verouderd (runtime gebruikt altijd "main")
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Details van sessievelden">

- **`scope`**: basisstrategie voor het groeperen van sessies in groepschatcontexten.
  - `per-sender` (standaard): elke afzender krijgt een geïsoleerde sessie binnen een kanaalcontext.
  - `global`: alle deelnemers binnen een kanaalcontext delen één sessie (alleen gebruiken wanneer een gedeelde context de bedoeling is).
- **`dmScope`**: hoe privéberichten worden gegroepeerd.
  - `main`: alle privéberichten delen de hoofdsessie.
  - `per-peer`: isoleren op afzender-id over kanalen heen.
  - `per-channel-peer`: isoleren per kanaal + afzender (aanbevolen voor inboxen met meerdere gebruikers).
  - `per-account-channel-peer`: isoleren per account + kanaal + afzender (aanbevolen voor meerdere accounts).
- **`identityLinks`**: canonieke id's koppelen aan peers met een providervoorvoegsel om sessies tussen kanalen te delen. Dock-opdrachten zoals `/dock_discord` gebruiken dezelfde koppeling om de antwoordroute van de actieve sessie over te schakelen naar een andere gekoppelde kanaalpeer; zie [Kanalen docken](/nl/concepts/channel-docking).
- **`reset`**: primair resetbeleid. `none` schakelt automatische resets uit en is de standaard; Compaction begrenst in plaats daarvan de actieve context. `daily` reset om `atHour` lokale tijd; `idle` reset na `idleMinutes`. Wanneer beide zijn geconfigureerd, geldt degene die het eerst verloopt. `/new` en `/reset` blijven in elke modus beschikbaar. De actualiteit voor dagelijkse resets gebruikt `sessionStartedAt` van de sessierij; de actualiteit voor resets wegens inactiviteit gebruikt `lastInteractionAt`. Schrijfacties van achtergrond-/systeemgebeurtenissen, zoals Heartbeat, Cron-wake-ups, exec-meldingen en Gateway-boekhouding, kunnen `updatedAt` bijwerken, maar houden dagelijkse/inactieve sessies niet actueel.
  - **`resetByType`**: overschrijvingen per type (`direct`, `group`, `thread`). Doctor migreert verouderde `dm`-vermeldingen naar `direct`; het schema weigert `dm`.
- **`resetByChannel`**: overschrijvingen van resets per kanaal, met provider-/kanaal-id als sleutel. Wanneer het kanaal van de sessie een overeenkomende vermelding heeft, heeft deze voor die sessie zonder meer voorrang op `resetByType`/`reset`. Alleen gebruiken wanneer één kanaal ander resetgedrag nodig heeft dan het beleid op typeniveau.
- **`mainKey`**: verouderd veld. De runtime gebruikt altijd `"main"` voor de hoofdbucket voor directe chats.
- **`sendPolicy`**: vergelijken op `channel`, `chatType` (`direct|group|channel`, met verouderde alias `dm`), `keyPrefix` of `rawKeyPrefix`. De eerste weigering geldt.
- **`maintenance`**: besturingselementen voor opschoning en bewaring van de sessieopslag.
  - `mode`: `enforce` voert opschoning uit en is de standaard; `warn` geeft alleen waarschuwingen.
  - `pruneAfter`: leeftijdsgrens voor verouderde vermeldingen (standaard `30d`).
  - `maxEntries`: maximaal aantal SQLite-sessievermeldingen (standaard `500`). Tijdens runtimeschrijfacties wordt opschoning in batches uitgevoerd met een kleine hoogwaterbuffer voor limieten van productieomvang; `openclaw sessions cleanup --enforce` past de limiet onmiddellijk toe.
  - Kortlevende Gateway-probesessies voor modelruns gebruiken een vaste bewaartermijn van `24h`, maar de opschoning is afhankelijk van druk: verouderde rijen voor strikte modelrunprobes worden alleen verwijderd wanneer onderhoud van sessievermeldingen of limietdruk wordt bereikt. Alleen strikte, expliciete probesleutels die overeenkomen met `agent:*:explicit:model-run-<uuid>` komen in aanmerking; normale directe, groeps-, thread-, Cron-, hook-, Heartbeat-, ACP- en subagentsessies nemen deze bewaartermijn van 24 uur niet over. Wanneer de opschoning van modelruns wordt uitgevoerd, gebeurt dit vóór de bredere opschoning van verouderde vermeldingen via `pruneAfter` en vóór de limiet van `maxEntries`.
  - Het verouderde `rotateBytes` wordt door het huidige schema geweigerd; `openclaw doctor --fix` verwijdert het uit oudere configuraties.
  - `resetArchiveRetention`: op leeftijd gebaseerde bewaring voor archieven van geresette/verwijderde transcripten. Standaard blijven archieven bestaan totdat ze wegens het schijfbudget worden verwijderd; stel een duur in om verwijdering op basis van verstreken tijd in te schakelen, of `false` om dit expliciet uit te schakelen.
  - `maxDiskBytes`: optioneel schijfbudget voor de sessiemap. In de modus `warn` worden waarschuwingen gelogd; in de modus `enforce` worden de oudste artefacten/sessies het eerst verwijderd.
  - `highWaterBytes`: optioneel doel na opschoning op basis van het budget. Standaard `80%` van `maxDiskBytes`.
- **`threadBindings`**: algemene standaardwaarden voor sessiefuncties die aan threads zijn gebonden.
  - `enabled`: hoofdschakelaar voor ondersteunde kanaal-threadkoppelingen
  - `idleHours`: standaard automatisch verlies van focus na inactiviteit, in uren (`0` schakelt dit uit; providers kunnen dit overschrijven)
  - `maxAgeHours`: standaard absolute maximumleeftijd in uren (`0` schakelt dit uit; providers kunnen dit overschrijven)
  - `spawnSessions`: standaardpoort voor het aanmaken van threadgebonden werksessies vanuit `sessions_spawn` en door ACP gestarte threads. Standaard `true` wanneer threadkoppelingen zijn ingeschakeld; providers/accounts kunnen dit overschrijven.
  - `defaultSpawnContext`: standaard native subagentcontext voor threadgebonden starts (`"fork"` of `"isolated"`). Standaard `"fork"`.
- **`sharing`**: bepaalt welke samenwerkingsmodi per sessie eigenaren en `operator.admin`-verbindingen mogen selecteren. Elke vlag is standaard `true`; door een vlag op `false` in te stellen, wordt die keuze uit de Control UI verwijderd en wordt deze bij zichtbaarheid tijdens het aanmaken of door `session.visibility.set` geweigerd. Nieuwe sessies beginnen als `shared`, tenzij de Control UI er een als concept start.
  - `readOnly`: `read-only` toestaan, waarbij niet-leden kunnen meekijken maar geen berichten kunnen verzenden, bijsturen, afbreken, goedkeuren of de sessiestatus wijzigen.
  - `suggest`: `suggest` toestaan. In deze fase dwingt dit hetzelfde toelatingsgedrag af als `read-only`; de wachtrij voor suggesties is een latere functie.
  - `drafts`: `draft` toestaan, waarmee de sessie wordt verborgen in sessielijsten en gebeurtenisuitzendingen voor iedereen die geen beheerder of eigenaar is.

Wijzigingen in lidmaatschap en zichtbaarheid worden als systeemnotities in het sessietranscript geschreven. Deze besturingselementen coördineren operators die één agent delen; ze vormen geen beveiligingsgrens tussen tenants. Gebruik afzonderlijke Gateways of agents wanneer het werk isolatie vereist.

</Accordion>

---

## Berichten

```json5
{
  messages: {
    responsePrefix: "🦞", // of "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all | off | none
    queue: {
      mode: "steer", // steer (standaard) | followup | collect | interrupt
      debounceMs: 500,
      cap: 20,
      drop: "summarize", // old | new | summarize (standaard)
      byChannel: {
        whatsapp: "followup",
        telegram: "followup",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 schakelt dit uit
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Antwoordvoorvoegsel

Overschrijvingen per kanaal/account: `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Resolutie (de meest specifieke geldt): account → kanaal → algemeen. `""` schakelt dit uit en stopt de cascade. `"auto"` leidt `[{identity.name}]` af.

**Sjabloonvariabelen:**

| Variabele          | Beschrijving            | Voorbeeld                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | Korte modelnaam       | `claude-opus-4-6`           |
| `{modelFull}`     | Volledige model-id  | `anthropic/claude-opus-4-6` |
| `{provider}`      | Providernaam          | `anthropic`                 |
| `{thinkingLevel}` | Huidig denkniveau | `high`, `low`, `off`        |
| `{identity.name}` | Naam van agentidentiteit    | (hetzelfde als `"auto"`)          |

Variabelen zijn niet hoofdlettergevoelig. `{think}` is een alias voor `{thinkingLevel}`.

### Bevestigingsreactie

- Standaard de `identity.emoji` van de actieve agent, anders `"👀"`. Stel `""` in om dit uit te schakelen.
- Overschrijvingen per kanaal: `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Resolutievolgorde: account → kanaal → `messages.ackReaction` → terugval op identiteit.
- Bereik: `group-mentions` (standaard), `group-all`, `direct`, `all` of `off`/`none` (schakelt bevestigingsreacties volledig uit).
- `messages.statusReactions.enabled`: schakelt reacties voor levenscyclusstatussen in op Slack, Discord, Signal, Telegram en WhatsApp.
  Op Discord blijven statusreacties ingeschakeld wanneer bevestigingsreacties actief zijn als dit niet is ingesteld.
  Op Slack, Signal, Telegram en WhatsApp moet dit expliciet op `true` worden ingesteld om reacties voor levenscyclusstatussen in te schakelen.
  Slack gebruikt standaard de native assistentstatus voor threads en roterende laadberichten om voortgang aan te geven, terwijl de geconfigureerde bevestigingsreactie statisch blijft.

### Wachtrij

- `mode`: wachtrijstrategie voor inkomende berichten die binnenkomen terwijl een sessierun actief is. Standaard: `"steer"`.
  - `steer`: de nieuwe prompt in de actieve run invoegen.
  - `followup`: de nieuwe prompt uitvoeren nadat de actieve run is voltooid.
  - `collect`: compatibele berichten bundelen en later samen uitvoeren.
  - `interrupt`: de actieve run afbreken voordat de nieuwste prompt wordt gestart.
- `debounceMs`: vertraging voordat een bericht uit de wachtrij of een bijgestuurd bericht wordt verzonden. Standaard: `500`.
- `cap`: maximaal aantal berichten in de wachtrij voordat het verwijderingsbeleid wordt toegepast. Standaard: `20`.
- `drop`: strategie wanneer de limiet wordt overschreden. `"summarize"` (standaard) verwijdert de oudste vermeldingen maar bewaart compacte samenvattingen; `"old"` verwijdert de oudste zonder samenvattingen; `"new"` weigert het nieuwste item.
- `byChannel`: overschrijvingen per kanaal voor `mode`, met provider-id als sleutel.
- `debounceMsByChannel`: overschrijvingen per kanaal voor `debounceMs`, met provider-id als sleutel.

### Debounce voor inkomende berichten

Bundelt snel opeenvolgende berichten met alleen tekst van dezelfde afzender tot één agentbeurt. Media/bijlagen worden onmiddellijk verwerkt. Besturingsopdrachten omzeilen debouncing. Standaard `debounceMs`: `2000`.

### Overige berichtsleutels

- `channels.whatsapp.responsePrefix`: voorvoegsel voor uitgaande WhatsApp-antwoorden. Doctor verplaatst de verouderde inkomende waarde `messagePrefix` alleen hierheen wanneer deze canonieke waarde niet is ingesteld.
- `messages.visibleReplies`: bepaalt zichtbare bronantwoorden in directe, groeps- en kanaalgesprekken (`"message_tool"` vereist `message(action=send)` voor zichtbare uitvoer; `"automatic"` plaatst normale antwoorden zoals voorheen).
- `messages.usageTemplate` / `messages.responseUsage`: aangepast `/usage`-voettekstsjabloon en standaard gebruiksmodus per antwoord (`off | tokens | full`, plus de verouderde alias `on` voor `tokens`).
- `messages.groupChat.mentionPatterns` / `historyLimit`: triggers voor vermeldingen in groepsberichten en de grootte van het geschiedenisvenster.
- `messages.suppressToolErrors`: wanneer `true`, worden waarschuwingen voor `⚠️`-toolfouten die aan de gebruiker worden getoond onderdrukt (de agent ziet fouten nog steeds in de context en kan het opnieuw proberen). Standaard: `false`.

### TTS (tekst-naar-spraak)

```json5
{
  tts: {
    auto: "off", // off (standaard) | altijd | inkomend | getagd
    mode: "final", // definitief | alles
    provider: "elevenlabs",
    summaryModel: "openai/gpt-5.4-mini",
    modelOverrides: { enabled: true },
    maxTextLength: 4000,
    timeoutMs: 30000,
    providers: {
      elevenlabs: {
        apiKey: "example-elevenlabs-api-key",
        baseUrl: "https://api.elevenlabs.io",
        speakerVoiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      microsoft: {
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
      },
      openai: {
        apiKey: "example-openai-api-key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        speakerVoice: "coral",
      },
    },
  },
}
```

Het globale voorkeurenpad is de machinestatus (standaard
`~/.openclaw/settings/tts.json`; overschrijf met `OPENCLAW_TTS_PREFS`). Geavanceerde
multi-agentconfiguraties kunnen `agents.entries.<id>.tts.prefsPath` instellen voor afzonderlijke
voorkeursopslagen per agent.

- `auto` bepaalt de standaardmodus voor automatische TTS: `off`, `always`, `inbound` of `tagged`. `/tts on|off` kan lokale voorkeuren overschrijven en `/tts status` toont de effectieve status.
- `summaryModel` overschrijft `agents.defaults.model.primary` voor automatische samenvattingen.
- `modelOverrides` is standaard ingeschakeld (`enabled !== false`); `modelOverrides.allowProvider` moet expliciet worden ingeschakeld.
- API-sleutels vallen terug op `ELEVENLABS_API_KEY`/`XI_API_KEY` en `OPENAI_API_KEY`.
- Gebundelde spraakproviders zijn eigendom van plugins. Als `plugins.allow` is ingesteld, neem dan elke TTS-providerplugin op die je wilt gebruiken, bijvoorbeeld `microsoft` voor Edge TTS. De verouderde provider-id `edge` wordt geaccepteerd als alias voor `microsoft`.
- `providers.openai.baseUrl` overschrijft het OpenAI TTS-eindpunt. De resolutievolgorde is configuratie, vervolgens `OPENAI_TTS_BASE_URL` en daarna `https://api.openai.com/v1`.
- Wanneer `providers.openai.baseUrl` naar een niet-OpenAI-eindpunt verwijst, behandelt OpenClaw dit als een OpenAI-compatibele TTS-server en versoepelt het de validatie van model en stem.

---

## Praten

Standaardwaarden voor de praatmodus (macOS/iOS/Android en de Control UI in de browser).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_multilingual_v2",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    consultThinkingLevel: "low",
    consultFastMode: true,
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Spreek hartelijk en houd antwoorden kort.",
      mode: "realtime", // realtime | spraak-naar-tekst-naar-spraak | transcriptie
      transport: "webrtc", // webrtc | provider-websocket | gateway-relay | beheerde-ruimte
      vadThreshold: 0.5,
      silenceDurationMs: 500,
      prefixPaddingMs: 300,
      reasoningEffort: "medium",
      brain: "agent-consult", // agent-consult | directe-tools | geen
    },
  },
}
```

- `talk.provider` moet overeenkomen met een sleutel in `talk.providers` wanneer meerdere praatproviders zijn geconfigureerd.
- Verouderde platte praatsleutels (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) dienen alleen voor compatibiliteit. Voer `openclaw doctor --fix` uit om de opgeslagen configuratie te herschrijven naar `talk.providers.<provider>`.
- Stem-id's vallen terug op `ELEVENLABS_VOICE_ID` of `SAG_VOICE_ID` (gedrag van de macOS-praatclient).
- `providers.*.apiKey` accepteert tekenreeksen met platte tekst of SecretRef-objecten.
- Terugvallen op `ELEVENLABS_API_KEY` is alleen van toepassing wanneer er geen API-sleutel voor Praten is geconfigureerd.
- `providers.*.voiceAliases` maakt het mogelijk dat praatinstructies gebruiksvriendelijke namen gebruiken.
- `providers.mlx.modelId` selecteert de Hugging Face-repository die door de lokale MLX-helper van macOS wordt gebruikt. Indien weggelaten, gebruikt macOS `mlx-community/Soprano-80M-bf16`.
- MLX-weergave op macOS verloopt via de gebundelde helper `openclaw-mlx-tts` als die aanwezig is, of via een uitvoerbaar bestand op `PATH`; `OPENCLAW_MLX_TTS_BIN` overschrijft het helperpad voor ontwikkeling.
- `consultThinkingLevel` bepaalt het denkniveau voor de volledige OpenClaw-agentuitvoering achter realtime `openclaw_agent_consult`-aanroepen van Praten in de Control UI. Laat dit oningesteld om normaal sessie-/modelgedrag te behouden.
- `consultFastMode` stelt een eenmalige overschrijving van de snelle modus in voor realtime raadplegingen van Praten in de Control UI, zonder de normale instelling voor snelle modus van de sessie te wijzigen.
- `speechLocale` stelt de BCP 47-locale-id in die door Android, iOS en macOS wordt gebruikt voor spraakherkenning in Praten. Android gebruikt ook de taalcomponent ervan als leidraad voor realtime transcriptie van invoer. Laat dit oningesteld om de standaardwaarde van het apparaat te gebruiken.
- `silenceTimeoutMs` bepaalt hoelang de praatmodus na stilte van de gebruiker wacht voordat het transcript wordt verzonden. Ongedefinieerd behoudt het standaardpauzevenster van het platform (`700 ms on macOS and Android, 900 ms on iOS`).
- `realtime.instructions` voegt systeeminstructies voor de provider toe aan de ingebouwde realtimeprompt van OpenClaw, zodat de stemstijl kan worden geconfigureerd zonder de standaardrichtlijnen van `openclaw_agent_consult` te verliezen.
- `realtime.vadThreshold` stelt de drempelwaarde voor spraakactiviteit van de provider in van `0` (meest gevoelig) tot `1` (minst gevoelig). Ongedefinieerd behoudt de standaardwaarde van de provider.
- `realtime.silenceDurationMs` stelt het positieve gehele aantal voor het stiltevenster in voordat de provider een realtime gebruikersbeurt vastlegt. Ongedefinieerd behoudt de standaardwaarde van de provider.
- `realtime.prefixPaddingMs` stelt het niet-negatieve gehele aantal aan audio in dat wordt bewaard voordat gedetecteerde spraak begint. Ongedefinieerd behoudt de standaardwaarde van de provider.
- `realtime.reasoningEffort` stelt het providerspecifieke redeneerniveau voor realtime sessies in. Ongedefinieerd behoudt de standaardwaarde van de provider.
- `realtime.consultRouting`: `"provider-direct"` (standaard) behoudt rechtstreekse antwoorden van de provider wanneer de realtimeprovider een definitief gebruikerstranscript produceert zonder `openclaw_agent_consult`. `"force-agent-consult"` routeert het voltooide verzoek in plaats daarvan via OpenClaw.

---

## Gerelateerd

- [Configuratiereferentie](/nl/gateway/configuration-reference) — alle overige configuratiesleutels
- [Configuratie](/nl/gateway/configuration) — algemene taken en snelle configuratie
- [Configuratievoorbeelden](/nl/gateway/configuration-examples)
