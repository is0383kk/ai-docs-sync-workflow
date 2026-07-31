---
read_when:
    - '`tools.*`-beleid, toelatingslijsten of experimentele functies configureren'
    - Aangepaste providers registreren of basis-URL's overschrijven
    - OpenAI-compatibele zelfgehoste eindpunten instellen
sidebarTitle: Tools and custom providers
summary: Configuratie van tools (beleid, experimentele schakelaars, door providers ondersteunde tools) en aangepaste provider-/basis-URL-configuratie
title: Configuratie — tools en aangepaste providers
x-i18n:
    generated_at: "2026-07-27T04:59:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*`-configuratiesleutels en aangepaste provider-/basis-URL-instellingen. Zie [Configuratiereferentie](/nl/gateway/configuration-reference) voor agents, kanalen en andere configuratiesleutels op het hoogste niveau.

## Tools

### Toolprofielen

`tools.profile` stelt een basislijst met toegestane items in vóór `tools.allow`/`tools.deny`:

<Note>
Bij lokale onboarding wordt voor nieuwe lokale configuraties standaard `tools.profile: "coding"` gebruikt wanneer dit niet is ingesteld (bestaande expliciete profielen blijven behouden).
</Note>

| Profiel     | Omvat                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | alleen `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`, `image`, `image_generate`, `music_generate`, `video_generate`                |
| `messaging` | `group:messaging`, `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `ask_user` |
| `full`      | Geen beperking (hetzelfde als niet ingesteld)                                                                                                                                                                                                                          |

`coding` en `messaging` staan impliciet ook `bundle-mcp` toe (geconfigureerde MCP-servers).

### Toolgroepen

| Groep              | Tools                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` wordt geaccepteerd als alias voor `exec`)                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | Alle bovenstaande ingebouwde tools behalve `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` (exclusief plugintools)                                                                                                                                  |
| `group:plugins`    | Tools die eigendom zijn van geladen plugins, waaronder geconfigureerde MCP-servers die via `bundle-mcp` beschikbaar worden gesteld                                                                                                                                                           |

Met `spawn_task` kan een codingagent bevestigd vervolgwerk voorstellen zonder dit te starten. De Control UI toont de titel en samenvatting als een uitvoerbare chip; een door een Gateway ondersteunde TUI toont een gelijkwaardige interactieve prompt. Als een van beide wordt geaccepteerd, wordt een nieuwe beheerde worktree-sessie aangemaakt en wordt de volledige prompt daarheen verzonden terwijl de huidige beurt doorgaat. `dismiss_task` trekt een nog openstaande suggestie in aan de hand van de tijdelijke `task_id` die door `spawn_task` is geretourneerd.

De tools worden alleen aangeboden wanneer het initiërende operatoroppervlak Gateway-gebeurtenissen voor taaksuggesties kan ontvangen en uitvoeren. Kanaalsessies en lokale/ingebedde TUI-sessies ontvangen deze niet; kanaaltransporten hebben een overdraagbare getypeerde taakactie nodig voordat ze deze flow veilig beschikbaar kunnen stellen. Suggesties zijn proceslokaal en verdwijnen wanneer de Gateway opnieuw wordt gestart. Beide tools blijven opgenomen in het profiel `coding` en `group:sessions`, zodat de normale beleidsconfiguratie van `tools.allow` en `tools.deny` ze automatisch configureert wanneer het oppervlak ze ondersteunt.

### MCP- en plugintools binnen het toolbeleid van de sandbox

Geconfigureerde MCP-servers worden beschikbaar gesteld als tools die eigendom zijn van de plugin met plugin-id `bundle-mcp`. Normale toolprofielen kunnen ze toestaan, maar `tools.sandbox.tools` vormt een aanvullende controle voor sessies in een sandbox. Als de sandboxmodus `"all"` of `"non-main"` is, neem dan een van deze vermeldingen op in de lijst met toegestane sandboxtools wanneer MCP-/plugintools zichtbaar moeten zijn:

- `bundle-mcp` voor door OpenClaw beheerde MCP-servers uit `mcp.servers`
- de plugin-id voor een specifieke native plugin
- `group:plugins` voor alle geladen tools die eigendom zijn van plugins
- exacte toolnamen of server-globs van MCP-servers, zoals `outlook__send_mail` of `outlook__*`, wanneer je slechts één server wilt

Server-globs gebruiken het providerveilige MCP-servervoorvoegsel, niet noodzakelijkerwijs de onbewerkte sleutel `mcp.servers`. Niet-`[A-Za-z0-9_-]`-tekens worden `-`, namen die niet met een letter beginnen krijgen het voorvoegsel `mcp-`, en lange of dubbele voorvoegsels kunnen worden afgekapt of van een achtervoegsel worden voorzien; `mcp.servers["Outlook Graph"]` gebruikt bijvoorbeeld een glob zoals `outlook-graph__*`.

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

Zonder die vermelding op sandboxniveau kan de MCP-server nog steeds met succes worden geladen, terwijl de tools ervan vóór de provideraanvraag worden uitgefilterd. Gebruik `openclaw doctor` om deze situatie te detecteren voor door OpenClaw beheerde servers in `mcp.servers`. MCP-servers die uit gebundelde pluginmanifests of Claude `.mcp.json` worden geladen, gebruiken dezelfde sandboxcontrole, maar deze diagnose inventariseert die bronnen nog niet; gebruik dezelfde vermeldingen in de lijst met toegestane items als hun tools tijdens beurten in een sandbox verdwijnen.

### `tools.codeMode`

`tools.codeMode` schakelt het generieke code-mode-oppervlak van OpenClaw in. Wanneer dit is ingeschakeld
voor een uitvoering met tools, worden normale OpenClaw-tools achter de `tools.*`-catalogusbridge
in de sandbox geplaatst en zijn MCP-tools beschikbaar via de gegenereerde `MCP`-
naamruimte. Het model ziet normaal gesproken `exec` en `wait`; tools zoals `computer`
waarvan de gestructureerde resultaten niet via de uitsluitend voor JSON bestemde bridge kunnen worden doorgegeven, blijven rechtstreeks beschikbaar.

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

De verkorte notatie wordt ook geaccepteerd:

```json5
{
  tools: { codeMode: true },
}
```

MCP-declaraties worden in code mode beschikbaar gesteld via het alleen-lezen virtuele API-bestandsoppervlak.
Gastcode kan `API.list("mcp")` en
`API.read("mcp/<server>.d.ts")` aanroepen om TypeScript-achtige signaturen te inspecteren voordat
`MCP.<server>.<tool>()` wordt aangeroepen. Zie [Code Mode](/tools/code-mode) voor het
runtimecontract, de limieten en de stappen voor foutopsporing.

### `tools.allow` / `tools.deny`

Globaal beleid voor het toestaan/weigeren van tools (weigeren heeft voorrang). Niet hoofdlettergevoelig, ondersteunt `*`-jokertekens. Wordt ook toegepast wanneer de Docker-sandbox is uitgeschakeld.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` en `apply_patch` zijn afzonderlijke tool-id's. `allow: ["write"]` schakelt voor compatibele modellen ook `apply_patch` in, maar `deny: ["write"]` weigert `apply_patch` niet. Weiger `group:fs` of vermeld elke muterende tool expliciet om alle bestandswijzigingen te blokkeren:

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` en `alsoAllow` kunnen niet beide binnen hetzelfde bereik worden ingesteld (`tools`, `tools.byProvider.<id>`, `agents.entries.*.tools`) — de configuratievalidatie wijst dit af. Voeg vermeldingen uit `alsoAllow` samen in `allow`, of verwijder `allow` en gebruik in plaats daarvan `profile` + `alsoAllow`.
</Note>

### `tools.byProvider`

Beperk tools verder voor specifieke providers of modellen. Volgorde: basisprofiel → providerprofiel → toestaan/weigeren.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

Beperkt tools voor de oorspronkelijke aanvrager van de huidige beurt. Dit biedt extra beveiliging boven op de toegangscontrole van het kanaal; afzenderwaarden moeten afkomstig zijn van de kanaaladapter, niet uit de berichttekst. Hiermee wordt andere inhoud in de modelprompt niet geverifieerd; zie [Aanvragergebonden besturingselementen en promptcontext](/nl/gateway/security#requester-scoped-controls-and-prompt-context).

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

Sleutels gebruiken expliciete voorvoegsels: `channel:<channelId>:<senderId>`, `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` of `"*"`. Kanaal-id's zijn canonieke OpenClaw-id's; aliassen zoals `teams` worden genormaliseerd naar `msteams`. Verouderde sleutels zonder voorvoegsel worden uitsluitend geaccepteerd als `id:`. De overeenkomstvolgorde is kanaal+id, id, e164, gebruikersnaam, naam en vervolgens jokerteken.

`agents.entries.*.tools.toolsBySender` per agent overschrijft de globale afzenderovereenkomst wanneer deze overeenkomt, zelfs met een leeg `{}`-beleid.

### `tools.elevated`

Bepaalt verhoogde toegang tot exec buiten de sandbox:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- De overschrijving per agent (`agents.entries.*.tools.elevated`) kan uitsluitend verdere beperkingen opleggen.
- `/elevated on|off|ask|full` slaat de status per sessie op; inline richtlijnen gelden voor één bericht.
- Verhoogde `exec` omzeilt de sandbox en gebruikt het geconfigureerde ontsnappingspad (standaard `gateway`, of `node` wanneer het exec-doel `node` is).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

De weergegeven waarden zijn standaardwaarden, behalve `applyPatch.allowModels` (standaard leeg/niet ingesteld, wat betekent dat elk compatibel model `apply_patch` mag gebruiken). `approvalRunningNoticeMs` geeft een melding dat het proces nog actief is wanneer exec met goedkeuring lang duurt; `0` schakelt dit uit.

### `tools.loopDetection`

Veiligheidscontroles voor toollussen zijn **standaard uitgeschakeld**. Stel `enabled: true` in om detectie te activeren. Instellingen kunnen globaal worden gedefinieerd in `tools.loopDetection` en per agent worden overschreven via `agents.entries.*.tools.loopDetection`.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // or BRAVE_API_KEY env (Brave provider)
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // optional; omit for auto-detect
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

De weergegeven waarden zijn standaardwaarden, behalve `provider` en `userAgent`. `maxResponseBytes` begrenst tot 32000–10000000; `maxChars` begrenst tot `maxCharsCap` (verhoog `maxCharsCap` om grotere antwoorden toe te staan).

### `tools.media`

Configureert het begrijpen van binnenkomende media (afbeelding/audio/video):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` is de enige geconfigureerde modellenlijst. Elke vermelding declareert welke mogelijkheden deze verwerkt. De optionele `preferredModel`-selector accepteert `provider/model`, een model-id, `provider:<id>` voor vermeldingen met de standaardwaarde van de provider, of `cli:command`; overeenkomende vermeldingen worden vooraan geplaatst in de terugvalvolgorde van die mogelijkheid. Prompts, limieten, aanvraaginstellingen, bereik, bijlagebeleid en het herhalen van audiotranscripten per mogelijkheid behouden hun standaardwaarden voor geconfigureerde en automatisch gedetecteerde modellen; een modelvermelding kan modelspecifieke velden overschrijven.

<AccordionGroup>
  <Accordion title="Velden van een mediamodelvermelding">
    **Providervermelding** (`type: "provider"` of weggelaten):

    - `provider`: API-provider-id (`openai`, `anthropic`, `google`/`gemini`, `groq`, enz.)
    - `model`: overschrijving van model-id
    - `profile` / `preferredProfile`: selectie van `auth-profiles.json`-profiel

    **CLI-vermelding** (`type: "cli"`):

    - `command`: uit te voeren programma
    - `args`: argumenten met sjablonen (ondersteunt `{{AttachmentPath}}`, `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{Prompt}}`, `{{MaxChars}}`, enz.; `openclaw doctor --fix` migreert verouderde `{input}`-plaatsaanduidingen naar `{{AttachmentPath}}`). De oudere aliassen `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` en `{{MediaDir}}` blijven tijdens hun compatibiliteitsperiode beschikbaar, maar zijn verouderd.

    **Gemeenschappelijke velden:**

    - `capabilities`: lijst met een of meer van `image`, `audio` en `video`.
    - `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: overschrijvingen per vermelding.
    - Overeenkomende `timeoutSeconds`-vermeldingen voor afbeeldingsmodellen zijn ook van toepassing wanneer de agent de expliciete `image`-tool aanroept. Voor het begrijpen van afbeeldingen geldt deze time-out voor de aanvraag zelf en wordt deze niet verminderd door eerder voorbereidingswerk.
    - Bij fouten wordt teruggevallen op de volgende vermelding.

    Providerverificatie volgt de standaardvolgorde: `auth-profiles.json` → omgevingsvariabelen → `models.providers.*.apiKey`.

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

Bepaalt op welke sessies de sessietools (`sessions_list`, `sessions_history`, `sessions_send`) zich kunnen richten.

Standaard: `tree` (huidige sessie + hierdoor gestarte sessies, zoals subagents, plus op de achtergrond
gevolgde groepssessies voor dezelfde agent).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Zichtbaarheidsbereiken">
    - `self`: uitsluitend de sleutel van de huidige sessie.
    - `tree`: huidige sessie + sessies die door de huidige sessie zijn gestart (subagents). Voor leesbewerkingen omvat dit ook groepssessies van dezelfde agent die de huidige sessie volgt via groepsbewustzijn op de achtergrond.
    - `agent`: elke sessie die bij de huidige agent-id hoort (kan andere gebruikers omvatten als je sessies per afzender onder dezelfde agent-id uitvoert).
    - `all`: elke sessie. Voor doelen bij andere agents is nog steeds `tools.agentToAgent` vereist.
    - Sandboxbegrenzing: wanneer de huidige sessie in een sandbox draait en `agents.defaults.sandbox.sessionToolsVisibility="spawned"` (de standaardwaarde) is ingesteld, wordt de zichtbaarheid gedwongen op `tree`, zelfs als `tools.sessions.visibility="all"`.
    - Wanneer dit niet `all` is, bevat `sessions_list` een compact `visibility`-veld
      dat de effectieve modus beschrijft en waarschuwt dat sommige sessies buiten
      het huidige bereik mogelijk worden weggelaten.

  </Accordion>
</AccordionGroup>

Met de standaardwaarde `session.dmScope: "main"` maakt menselijke activiteit in een groep die groepssessie van
dezelfde agent op de achtergrond zichtbaar voor de hoofdsessie van de agent. In een configuratie met meerdere gebruikers deelt `"main"` bovendien
één DM-sessie tussen gebruikers, zodat elke daarheen gerouteerde gebruiker kan lezen uit op de achtergrond gevolgde groepen,
ook via `memory_search` van het sessiegeheugen. Gebruik een `dmScope` per peer voor DM-isolatie, of stel
`tools.sessions.visibility: "self"` in om leesbewerkingen uit gevolgde sessies op de achtergrond uit te schakelen.

### `tools.sessions_spawn`

Bepaalt ondersteuning voor inline bijlagen voor `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: set true to allow inline file attachments
        maxTotalBytes: 5242880, // 5 MB total across all files
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB per file
        retainOnSessionKeep: false, // keep attachments when cleanup="keep"
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Opmerkingen over bijlagen">
    - Voor bijlagen is `enabled: true` vereist.
    - Bijlagen van subagents worden in de onderliggende werkruimte opgeslagen op `.openclaw/attachments/<uuid>/` met een `.manifest.json`.
    - ACP-bijlagen zijn uitsluitend afbeeldingen en worden inline doorgestuurd naar de ACP-runtime nadat dezelfde limieten voor het aantal bestanden, het aantal bytes per bestand en het totale aantal bytes zijn doorstaan.
    - Bijlage-inhoud wordt automatisch geredigeerd bij het permanent opslaan van transcripten.
    - Base64-invoer wordt gevalideerd met strikte controles op alfabet en opvulling, plus een groottecontrole vóór het decoderen.
    - Bestandsmachtigingen voor bijlagen van subagents zijn `0700` voor mappen en `0600` voor bestanden.
    - Het opschonen van subagents volgt het `cleanup`-beleid: `delete` verwijdert bijlagen altijd; `keep` behoudt ze uitsluitend wanneer `retainOnSessionKeep: true`.

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

Experimentele ingebouwde toolvlaggen. Standaard uitgeschakeld, tenzij een automatische inschakelregel voor strikt agentische GPT-5 van toepassing is.

```json5
{
  tools: {
    experimental: {
      planTool: true, // enable experimental update_plan
    },
  },
}
```

- `planTool`: schakelt de gestructureerde `update_plan`-tool in voor het bijhouden van niet-triviaal werk met meerdere stappen.
- Standaard: `false`, tenzij `agents.defaults.embeddedAgent.executionContract` (of een overschrijving per agent) is ingesteld op `"strict-agentic"` voor een `openai`-provideruitvoering met een model-id uit de GPT-5-familie (dit omvat ook OpenAI Codex CLI-uitvoeringen, omdat Codex-verificatie en modelroutering onder de `openai`-provider vallen). Stel `true` in om de tool buiten dat bereik geforceerd in te schakelen, of `false` om deze zelfs voor strikt agentische GPT-5-uitvoeringen uitgeschakeld te houden.
- Wanneer dit is ingeschakeld, voegt de systeemprompt ook gebruiksrichtlijnen toe, zodat het model de tool uitsluitend gebruikt voor substantieel werk en maximaal één stap `in_progress` houdt.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: standaardmodel voor gestarte subagents. Indien weggelaten, nemen subagents het model van de aanroeper over.
- `allowAgents`: standaardtoelatingslijst van geconfigureerde doelagent-id's voor `sessions_spawn` wanneer de aanvragende agent geen eigen `subagents.allowAgents` instelt (`["*"]` = elk geconfigureerd doel; standaard: alleen dezelfde agent). Verouderde vermeldingen waarvan de agentconfiguratie is verwijderd, worden door `sessions_spawn` geweigerd en uit `agents_list` weggelaten; voer `openclaw doctor --fix` uit om ze op te ruimen.
- `maxConcurrent`: maximaal aantal gelijktijdige subagentruns. Standaard: `8`.
- `runTimeoutSeconds`: time-out (seconden) voor `sessions_spawn` wanneer de aanroeper geen eigen overschrijving doorgeeft. Standaard: `0` (geen time-out); de hierboven getoonde `900` is een veelgebruikte opt-inwaarde, niet de ingebouwde standaardwaarde.
- `announceTimeoutMs`: time-out per aanroep (milliseconden) voor afleverpogingen van Gateway-`agent`-aankondigingen. Standaard: `120000`. Tijdelijke nieuwe pogingen kunnen ervoor zorgen dat de totale wachttijd voor de aankondiging langer is dan één geconfigureerde time-out.
- `archiveAfterMinutes`: aantal minuten nadat een subagentsessie is voltooid voordat deze automatisch wordt gearchiveerd. Standaard: `60`; `0` schakelt automatisch archiveren uit.
- Toolbeleid per subagent: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Aangepaste providers en basis-URL's

Providerplugins publiceren hun eigen modelcatalogusrijen. Voeg aangepaste providers toe via `models.providers` in de configuratie of `~/.openclaw/agents/<agentId>/agent/models.json`.

Het configureren van een aangepaste/lokale provider-`baseUrl` is tevens de beperkte netwerkvertrouwensbeslissing voor HTTP-modelverzoeken: OpenClaw staat precies die `scheme://host:port`-oorsprong toe via het beveiligde fetch-pad, zonder een afzonderlijke configuratieoptie toe te voegen of andere privé-oorsprongen te vertrouwen.

```json5
{
  models: {
    mode: "merge", // samenvoegen (standaard) | vervangen
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | enz.
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Authenticatie en samenvoegingsprioriteit">
    - Gebruik `authHeader: true` + `headers` voor aangepaste authenticatiebehoeften.
    - Overschrijf de hoofdmap van de agentconfiguratie met `OPENCLAW_AGENT_DIR`.
    - Samenvoegingsprioriteit voor overeenkomende provider-id's:
      - Niet-lege `models.json`-`baseUrl`-waarden van de agent hebben voorrang.
      - Niet-lege `apiKey`-waarden van de agent hebben alleen voorrang wanneer die provider in de huidige configuratie-/authenticatieprofielcontext niet door SecretRef wordt beheerd.
      - Door SecretRef beheerde `apiKey`-waarden van de provider worden vernieuwd vanuit bronmarkeringen (`ENV_VAR_NAME` voor omgevingsverwijzingen, `secretref-managed` voor bestands-/uitvoeringsverwijzingen) in plaats van opgeloste geheimen permanent op te slaan.
      - Door SecretRef beheerde headerwaarden van de provider worden vernieuwd vanuit bronmarkeringen (`secretref-env:ENV_VAR_NAME` voor omgevingsverwijzingen, `secretref-managed` voor bestands-/uitvoeringsverwijzingen).
      - Lege of ontbrekende `apiKey`/`baseUrl` van de agent vallen terug op `models.providers` in de configuratie.
      - Voor overeenkomende model-`contextWindow`/`maxTokens` heeft de expliciete configuratiewaarde voorrang wanneer deze aanwezig en geldig is (een positief eindig getal); anders wordt de impliciete/gegenereerde cataloguswaarde gebruikt.
      - Overeenkomende model-`contextTokens` volgt dezelfde regel waarbij expliciet voorrang heeft en anders impliciet wordt gebruikt; gebruik dit om de effectieve context te beperken zonder de oorspronkelijke modelmetadata te wijzigen.
      - Catalogi van providerplugins worden als gegenereerde, door de Plugin beheerde catalogusfragmenten opgeslagen onder de Pluginstatus van de agent.
      - Gebruik `models.mode: "replace"` wanneer de configuratie `models.json` volledig moet herschrijven en het samenvoegen van door de Plugin beheerde catalogusfragmenten moet overslaan.
      - Het permanent opslaan van markeringen is bronauthoritatief: markeringen worden geschreven vanuit de actieve momentopname van de bronconfiguratie (vóór oplossing), niet vanuit opgeloste geheime runtimewaarden.

  </Accordion>
</AccordionGroup>

### Details van providervelden

<AccordionGroup>
  <Accordion title="Catalogus op het hoogste niveau">
    - `models.mode`: gedrag van de providercatalogus (`merge` of `replace`).
    - `models.providers`: kaart met aangepaste providers, geïndexeerd op provider-id.
      - Veilige bewerkingen: gebruik `openclaw config set models.providers.<id> '<json>' --strict-json --merge` of `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` voor additieve updates. `config set` weigert destructieve vervangingen tenzij je `--replace` doorgeeft.

  </Accordion>
  <Accordion title="Providerverbinding en authenticatie">
    - `models.providers.*.api`: verzoekadapter (`openai-completions`, `openai-responses`, `openai-chatgpt-responses`, `anthropic-messages`, `google-generative-ai`, `google-vertex`, `github-copilot`, `bedrock-converse-stream`, `ollama`, `azure-openai-responses`). Gebruik voor zelfgehoste `/v1/chat/completions`-backends zoals MLX, vLLM, SGLang en de meeste OpenAI-compatibele lokale servers `openai-completions`. Een aangepaste provider met `baseUrl` maar zonder `api` gebruikt standaard `openai-completions`; stel `openai-responses` alleen in wanneer de backend `/v1/responses` ondersteunt.
    - `models.providers.*.apiKey`: providerreferentie (geef de voorkeur aan SecretRef-/omgevingssubstitutie).
    - `models.providers.*.auth`: authenticatiestrategie (`api-key`, `token`, `oauth`, `aws-sdk`).
    - `models.providers.*.contextWindow`: standaard oorspronkelijke contextvenster voor modellen onder deze provider wanneer de modelvermelding `contextWindow` niet instelt.
    - `models.providers.*.contextTokens`: standaard effectieve runtimecontextlimiet voor modellen onder deze provider wanneer de modelvermelding `contextTokens` niet instelt.
    - `models.providers.*.maxTokens`: standaardlimiet voor uitvoertokens voor modellen onder deze provider wanneer de modelvermelding `maxTokens` niet instelt.
    - `models.providers.*.timeoutSeconds`: optionele time-out per provider voor HTTP-modelverzoeken in seconden, inclusief verbinding, headers, hoofdtekst en afhandeling van het afbreken van het volledige verzoek.
    - `models.providers.*.injectNumCtxForOpenAICompat`: injecteer voor Ollama + `openai-completions` `options.num_ctx` in verzoeken (standaard: `true`).
    - `models.providers.*.authHeader`: dwing indien vereist het transport van referenties af in de `Authorization`-header.
    - `models.providers.*.baseUrl`: basis-URL van de upstream-API.
    - `models.providers.*.headers`: extra statische headers voor proxy-/tenantroutering.

  </Accordion>
  <Accordion title="Overschrijvingen voor verzoektransport">
    `models.providers.*.request`: transportoverschrijvingen voor HTTP-verzoeken aan modelproviders.

    - `request.headers`: extra headers (samengevoegd met de standaardwaarden van de provider). Waarden accepteren SecretRef.
    - `request.auth`: overschrijving van de authenticatiestrategie. Modi: `"provider-default"` (gebruik de ingebouwde authenticatie van de provider), `"authorization-bearer"` (met `token`), `"header"` (met `headerName`, `value`, optioneel `prefix`).
    - `request.proxy`: overschrijving van de HTTP-proxy. Modi: `"env-proxy"` (gebruik de omgevingsvariabelen `HTTP_PROXY`/`HTTPS_PROXY`), `"explicit-proxy"` (met `url`). Beide modi accepteren een optioneel `tls`-subobject.
    - `request.tls`: TLS-overschrijving voor directe verbindingen. Velden: `ca`, `cert`, `key`, `passphrase` (alle accepteren SecretRef), `serverName`, `insecureSkipVerify`.
    - `request.allowPrivateNetwork`: sta wanneer `true` HTTP-verzoeken van modelproviders naar privé-, CGNAT- of vergelijkbare bereiken toe via de HTTP-fetchbeveiliging van de provider. Basis-URL's van aangepaste/lokale providers vertrouwen de exact geconfigureerde oorsprong al, met uitzondering van metadata-/link-local-oorsprongen, die zonder expliciete opt-in geblokkeerd blijven. Stel dit in op `false` om exact-oorsprongvertrouwen uit te schakelen. WebSocket gebruikt dezelfde `request` voor headers/TLS, maar niet die SSRF-fetchpoort. Standaard `false`.

  </Accordion>
  <Accordion title="Modelcatalogusvermeldingen">
    - `models.providers.*.models`: expliciete vermeldingen in de providermodelcatalogus.
    - `models.providers.*.models.*.input`: modelinvoermodaliteiten. Gebruik `["text"]` voor modellen die alleen tekst ondersteunen en `["text", "image"]` voor modellen met ingebouwde beeld-/zichtondersteuning. Afbeeldingsbijlagen worden alleen in agentbeurten ingevoegd wanneer het geselecteerde model als beeldgeschikt is gemarkeerd.
    - `models.providers.*.models.*.contextWindow`: metadata van het oorspronkelijke contextvenster van het model. Dit overschrijft `contextWindow` op providerniveau voor dat model.
    - `models.providers.*.models.*.contextTokens`: optionele runtimecontextlimiet. Dit overschrijft `contextTokens` op providerniveau; gebruik dit wanneer je een kleiner effectief contextbudget wilt dan de oorspronkelijke `contextWindow` van het model; `openclaw models list` toont beide waarden wanneer ze verschillen.

    #### Aangepaste declaraties van providermogelijkheden

    Providercatalogi beheren `compat` voor gebundelde en in de catalogus bekende modelroutes. Kopieer die vlaggen niet naar de configuratie: OpenClaw gebruikt de catalogusrij wanneer de geconfigureerde `api` en `baseUrl` die route nog steeds identificeren. `openclaw doctor --fix` verwijdert overeenkomende verouderde overschrijvingen en meldt afwijkende waarden ter beoordeling.

    Een `compat`-blok blijft ondersteund voor een werkelijk aangepaste provider, een aangepast model of een catalogusmodel dat naar een ander eindpunt wordt gerouteerd. Stel alleen mogelijkheden in die voor dat eindpunt zijn geverifieerd:

    | Sleutel voor aangepaste route | Runtimecontract |
    | --- | --- |
    | `supportsStore` | Accepteert het OpenAI-verzoekveld `store`. |
    | `supportsPromptCacheKey` | Accepteert OpenAI-sleutels voor promptcache-/sessieaffiniteit. |
    | `supportsDeveloperRole` | Accepteert `developer`-berichten in plaats van `system` te vereisen. |
    | `supportsReasoningEffort` | Accepteert een besturingselement voor redeneerinspanning. |
    | `supportsTemperature` | Accepteert `temperature` voor dit model en deze adapter. |
    | `supportsUsageInStreaming` | Geeft gebruiksmetadata uit in streamingreacties. |
    | `supportsTools` | Ondersteunt gestructureerde tool-/functieaanroepen. Stel `false` in om tools uit te schakelen. |
    | `supportsStrictMode` | Accepteert strikte toolschema's. |
    | `requiresStringContent` | Vereist berichtinhoud als onbewerkte tekenreeks voor Chat Completions. |
    | `strictMessageKeys` | Vereist dat uitgaande berichten alleen geaccepteerde sleutels bevatten. |
    | `visibleReasoningDetailTypes` | Benoemt typen detailblokken voor redeneringen die veilig in transcripties kunnen worden weergegeven. |
    | `supportedReasoningEfforts` | Vermeldt de door het eindpunt geaccepteerde redeneringslabels. |
    | `reasoningEffortMap` | Wijst OpenClaw-denklabels toe aan eindpuntspecifieke labels. |
    | `maxTokensField` | Selecteert `max_tokens` of `max_completion_tokens`. |
    | `thinkingFormat` | Selecteert het dialect van de redeneringspayload van het eindpunt. |
    | `requiresToolResultName` | Vereist een toolnaam in berichten met toolresultaten. |
    | `requiresAssistantAfterToolResult` | Vereist een assistentbericht na toolresultaten. |
    | `requiresThinkingAsText` | Speelt redeneringen opnieuw af als tekst in plaats van als gestructureerde inhoud. |
    | `requiresReasoningContentOnAssistantMessages` | Behoudt DeepSeek-achtige `reasoning_content` tijdens het opnieuw afspelen. |
    | `toolSchemaProfile` | Selecteert een door de provider gedefinieerd normalisatieprofiel voor toolschema's. |
    | `unsupportedToolSchemaKeywords` | Verwijdert benoemde JSON Schema-trefwoorden die door het eindpunt worden geweigerd. |
    | `toolCallArgumentsEncoding` | Selecteert de codering van argumenten voor toolaanroepen van het eindpunt. |
    | `requiresOpenAiAnthropicToolPayload` | Converteert toolaanroepen in OpenAI-vorm naar payloads van de Anthropic-familie. |

  </Accordion>
  <Accordion title="Amazon Bedrock-detectie">
    - `plugins.entries.amazon-bedrock.config.discovery`: hoofdinstelling voor automatische Bedrock-detectie.
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`: impliciete detectie in-/uitschakelen.
    - `plugins.entries.amazon-bedrock.config.discovery.region`: AWS-regio voor detectie.
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: optioneel provider-id-filter voor gerichte detectie.
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: pollinginterval voor het vernieuwen van de detectie.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: standaardcontextvenster voor gedetecteerde modellen.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: standaardmaximumaantal uitvoertokens voor gedetecteerde modellen.

  </Accordion>
</AccordionGroup>

Interactieve onboarding van aangepaste providers leidt ondersteuning voor afbeeldingsinvoer af uit bekende patronen voor vision-model-id's, waaronder GPT-4o/GPT-4.1/GPT-5+, de redeneringsfamilies `o1`/`o3`/`o4`, Claude, Gemini, elke id met het achtervoegsel `-vl` (Qwen-VL en vergelijkbare modellen) en benoemde families zoals LLaVA, Pixtral, InternVL, Mllama, MiniCPM-V en GLM-4V; voor bekende families met uitsluitend tekst (Llama, DeepSeek, Mistral/Mixtral, Kimi/Moonshot, Codestral, Devstral, Phi, QwQ, CodeLlama en kale Qwen-id's zonder het achtervoegsel vl/vision) wordt de extra vraag overgeslagen. Bij onbekende model-id's wordt nog steeds naar afbeeldingsondersteuning gevraagd. Niet-interactieve onboarding gebruikt dezelfde afleiding; geef `--custom-image-input` door om metadata met afbeeldingsondersteuning af te dwingen of `--custom-text-input` om metadata met uitsluitend tekst af te dwingen.

### Providervoorbeelden

<AccordionGroup>
  <Accordion title="Cerebras (GLM 4.7 / GPT OSS)">
    De officiële externe provider-Plugin `cerebras` kan dit configureren via `openclaw onboard --auth-choice cerebras-api-key`. Gebruik alleen een expliciete providerconfiguratie om de standaardwaarden te overschrijven.

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Gebruik `cerebras/zai-glm-4.7` voor Cerebras; `zai/glm-4.7` voor rechtstreeks gebruik van Z.AI.

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    Ingebouwde provider die compatibel is met Anthropic. Snelkoppeling: `openclaw onboard --auth-choice kimi-code-api-key`.

  </Accordion>
  <Accordion title="Lokale modellen (LM Studio)">
    Zie [Lokale modellen](/nl/gateway/local-models). Kort gezegd: voer op krachtige hardware een groot lokaal model uit via de LM Studio Responses API; behoud samengevoegde gehoste modellen als terugvaloptie.
  </Accordion>
  <Accordion title="MiniMax M3 (rechtstreeks)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    Stel `MINIMAX_API_KEY` in. Snelkoppelingen: `openclaw onboard --auth-choice minimax-global-api` of `openclaw onboard --auth-choice minimax-cn-api`. De modelcatalogus gebruikt standaard M3 en bevat ook de M2.7-varianten. Op het streamingpad dat compatibel is met Anthropic schakelt OpenClaw het denkproces van MiniMax M2.x standaard uit, tenzij je zelf expliciet `thinking` instelt; MiniMax-M3 (en M3.x) blijft standaard het pad voor weggelaten/adaptief denken van de provider gebruiken. `/fast on` of `params.fastMode: true` herschrijft `MiniMax-M2.7` naar `MiniMax-M2.7-highspeed`.

  </Accordion>
  <Accordion title="Moonshot AI (Kimi)">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    Voor het Chinese eindpunt: `baseUrl: "https://api.moonshot.cn/v1"` of `openclaw onboard --auth-choice moonshot-api-key-cn`.

    Eigen Moonshot-eindpunten geven aan dat ze compatibel zijn met streamingverbruik via het gedeelde transport `openai-completions`, en OpenClaw baseert dit op de mogelijkheden van het eindpunt in plaats van uitsluitend op de ingebouwde provider-id.

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    Stel `OPENCODE_API_KEY` (of `OPENCODE_ZEN_API_KEY`) in. Gebruik `opencode/...`-verwijzingen voor de Zen-catalogus of `opencode-go/...`-verwijzingen voor de Go-catalogus. Snelkoppeling: `openclaw onboard --auth-choice opencode-zen` of `openclaw onboard --auth-choice opencode-go`.

  </Accordion>
  <Accordion title="Synthetic (compatibel met Anthropic)">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    De basis-URL moet `/v1` weglaten (de Anthropic-client voegt dit toe). Snelkoppeling: `openclaw onboard --auth-choice synthetic-api-key`.

  </Accordion>
  <Accordion title="Z.AI (GLM-4.7)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    Stel `ZAI_API_KEY` in. Modelverwijzingen gebruiken de canonieke provider-id `zai/*`. Snelkoppeling: `openclaw onboard --auth-choice zai-api-key`.

    - Algemeen eindpunt: `https://api.z.ai/api/paas/v4`
    - Eindpunt voor programmeren: `https://api.z.ai/api/coding/paas/v4`
    - De standaardauthenticatiekeuze `zai-api-key` test je sleutel en detecteert automatisch bij welk eindpunt deze hoort (als de detectie geen uitsluitsel geeft, wordt teruggevallen op een vraag met Global als standaardwaarde). Voor expliciete selectie zijn ook afzonderlijke authenticatiekeuzes voor CN en Coding-Plan beschikbaar.
    - Definieer voor het algemene eindpunt een aangepaste provider met een overschrijving van de basis-URL.

  </Accordion>
</AccordionGroup>

---

## Gerelateerd

- [Configuratie — agents](/nl/gateway/config-agents)
- [Configuratie — kanalen](/nl/gateway/config-channels)
- [Configuratiereferentie](/nl/gateway/configuration-reference) — overige sleutels op het hoogste niveau
- [Tools en plugins](/nl/tools)
