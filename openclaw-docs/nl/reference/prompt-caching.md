---
read_when:
    - Je wilt de kosten voor prompttokens verlagen met cachebehoud
    - Je hebt cachegedrag per agent nodig in opstellingen met meerdere agents.
    - Je stemt Heartbeat en opschoning op basis van cache-TTL samen af
summary: Instellingen voor promptcaching, samenvoegvolgorde, providergedrag en optimalisatiepatronen
title: Promptcaching
x-i18n:
    generated_at: "2026-07-27T06:32:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99dfd3d226d37014110adf16818051236114dcb0277e9b4d13eaced0f1fc03aa
    source_path: reference/prompt-caching.md
    workflow: 16
---

Promptcaching stelt een modelprovider in staat om een ongewijzigd promptvoorvoegsel (systeem-/ontwikkelaarsinstructies, tooldefinities en andere stabiele context) in opeenvolgende beurten te hergebruiken in plaats van het bij elke aanvraag opnieuw te verwerken. Dit verlaagt de tokenkosten en latentie bij langlopende sessies met herhaalde context.

OpenClaw normaliseert het providergebruik naar `cacheRead` en `cacheWrite` wanneer de upstream-API deze tellers beschikbaar stelt. Gebruiksoverzichten (`/status` en vergelijkbare) vallen terug op de laatste gebruiksvermelding in het transcript wanneer de momentopname van de live sessie geen cachetellers bevat; een niet-nulwaarde uit de live sessie heeft altijd voorrang op de terugvalwaarde.

Providerreferenties:

- [Anthropic-promptcaching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [OpenAI-promptcaching](https://developers.openai.com/api/docs/guides/prompt-caching)

## Belangrijkste instellingen

### `cacheRetention`

Waarden: `"none" | "short" | "long"`. Configureerbaar als algemene standaardwaarde, per model en per agent.
`"standard"` is geen alias; gebruik `"short"` voor het standaardcachevenster van de provider. Ongeldige waarden worden met een waarschuwing genegeerd.

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # overschrijft de algemene standaardwaarde voor dit model
  list:
    - id: "alerts"
      params:
        cacheRetention: "none" # overschrijft beide standaardwaarden voor deze agent
```

Samenvoegvolgorde (later heeft voorrang):

1. `agents.defaults.params` - algemene standaardwaarde voor alle modellen
2. `agents.defaults.models["provider/model"].params` - overschrijving per model
3. `agents.entries.*.params` - overschrijving per agent, gekoppeld op basis van agent-id

Bron: `src/agents/embedded-agent-runner/extra-params.ts` (`resolveExtraParams`).

### `contextPruning.mode: "cache-ttl"`

Snoeit oude context met toolresultaten nadat het TTL-venster van de cache is verstreken, zodat een aanvraag na inactiviteit een te omvangrijke geschiedenis niet opnieuw in de cache plaatst.

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

Zie [Sessies snoeien](/nl/concepts/session-pruning) voor het volledige gedrag.

### Cache warm houden met Heartbeat

Heartbeat kan cachevensters warm houden en herhaalde cacheschrijfacties na perioden van inactiviteit beperken. Configureerbaar voor alle agents (`agents.defaults.heartbeat`) of per agent (`agents.entries.*.heartbeat`).

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

## Providergedrag

### Anthropic (directe API en Vertex AI)

- `cacheRetention` wordt ondersteund voor de providers `anthropic` en `anthropic-vertex`, en voor Claude-modellen op `amazon-bedrock` en aangepaste, met `anthropic-messages` compatibele eindpunten wanneer `cacheRetention` expliciet is ingesteld.
- Wanneer dit niet is ingesteld, vult OpenClaw `cacheRetention: "short"` vooraf in voor directe Anthropic (alleen de providers `anthropic` en `anthropic-vertex`; andere routes uit de Anthropic-familie vereisen een expliciete waarde).
- Native Anthropic Messages-antwoorden stellen `cache_read_input_tokens` en `cache_creation_input_tokens` beschikbaar, die worden toegewezen aan `cacheRead` en `cacheWrite`.
- `cacheRetention: "short"` wordt toegewezen aan de standaard tijdelijke cache van 5 minuten. `cacheRetention: "long"` vraagt om de TTL van 1 uur (`cache_control: { type: "ephemeral", ttl: "1h" }`) wanneer dit expliciet is ingesteld. Een impliciete of door de omgeving aangestuurde lange bewaartermijn (`OPENCLAW_CACHE_RETENTION=long` zonder expliciete `cacheRetention`) wordt alleen op `api.anthropic.com`- of Vertex AI-hosts (`aiplatform.googleapis.com` / `*-aiplatform.googleapis.com`) opgewaardeerd naar de TTL van 1 uur; andere hosts behouden de cache van 5 minuten.

Bron: `packages/ai/src/transports/anthropic-payload-policy.ts` (`resolveAnthropicEphemeralCacheControl`, `isLongTtlEligibleEndpoint`).

### OpenAI (directe API)

- Promptcaching verloopt automatisch bij ondersteunde recente modellen; OpenClaw voegt geen cachemarkeringen op blokniveau toe.
- OpenClaw verzendt `prompt_cache_key` om de cacheroutering in opeenvolgende beurten stabiel te houden. Directe `api.openai.com`-hosts krijgen dit automatisch. OpenAI-compatibele proxy's (oMLX, llama.cpp, aangepaste eindpunten) vereisen `compat.supportsPromptCacheKey: true` in de modelconfiguratie om dit in te schakelen; dit wordt nooit automatisch gedetecteerd voor een proxy.
- `prompt_cache_retention: "24h"` wordt alleen toegevoegd wanneer `cacheRetention: "long"` is geselecteerd en het gevonden eindpunt zowel de cachesleutel als lange bewaring ondersteunt (`compat.supportsLongCacheRetention`, standaard waar; compatibiliteitsprofielen van Together AI en Cloudflare schakelen dit uit). `cacheRetention: "none"` onderdrukt beide velden.
- Cachetreffers worden beschikbaar gesteld via `usage.prompt_tokens_details.cached_tokens` (Chat Completions) of `input_tokens_details.cached_tokens` (Responses API), die worden toegewezen aan `cacheRead`.
- Responses API-payloads kunnen ook `input_tokens_details.cache_write_tokens` beschikbaar stellen, dat wordt toegewezen aan `cacheWrite` en wordt berekend volgens het cacheschrijftarief van het model; bij Responses-payloads waarin het veld ontbreekt, blijft `cacheWrite` op `0`. De Chat Completions API van OpenAI documenteert of retourneert geen `cache_write_tokens`-teller, maar OpenClaw leest daar toch `prompt_tokens_details.cache_write_tokens` voor OpenRouter-compatibele en DeepSeek-achtige proxy's die een afzonderlijke schrijfteller rapporteren.
- In de praktijk gedraagt OpenAI zich meer als een cache voor een initieel voorvoegsel dan als Anthropic, dat de volledige voortschrijdende geschiedenis hergebruikt. Zie [Verwachtingen voor live OpenAI-gebruik](#openai-live-expectations) hieronder.

### Amazon Bedrock

- Anthropic Claude-modelreferenties (`amazon-bedrock/*anthropic.claude*`, plus de AWS-voorvoegsels voor systeeminferentieprofielen `us.`/`eu.`/`global.anthropic.claude*`) ondersteunen expliciete doorgifte van `cacheRetention`.
- Niet-Anthropic Bedrock-modellen (bijvoorbeeld `amazon.nova-*`) worden tijdens runtime zonder cachebewaring verwerkt, ongeacht een eventueel geconfigureerde waarde voor `cacheRetention`.
- Ondoorzichtige ARN's van Bedrock-toepassingsinferentieprofielen (profiel-id's die geen `claude` bevatten) worden eveneens zonder cachebewaring verwerkt, tenzij `cacheRetention` expliciet is ingesteld, omdat de modelfamilie niet alleen uit de ARN kan worden afgeleid.

### OpenRouter

Voor `openrouter/anthropic/*`-modelreferenties voegt OpenClaw Anthropic-`cache_control`-markeringen toe aan promptblokken voor het systeem en de ontwikkelaar, maar alleen wanneer de aanvraag nog steeds naar een geverifieerde OpenRouter-route gaat (`openrouter` op het standaardeindpunt, of een provider/basis-URL die wordt herleid tot `openrouter.ai`). Als het model naar een willekeurige OpenAI-compatibele proxy-URL wordt omgeleid, stopt deze toevoeging.

`contextPruning.mode: "cache-ttl"` is toegestaan voor de modelreferenties `openrouter/anthropic/*`, `openrouter/deepseek/*`, `openrouter/moonshot/*`, `openrouter/moonshotai/*` en `openrouter/zai/*`, omdat deze routes promptcaching aan providerzijde afhandelen zonder de door OpenClaw toegevoegde markeringen nodig te hebben.

Bron: `extensions/openrouter/index.ts` (`OPENROUTER_CACHE_TTL_MODEL_PREFIXES`).

Het opbouwen van een DeepSeek-cache op OpenRouter gebeurt naar beste vermogen en kan enkele seconden duren; een directe vervolgaanvraag kan nog steeds `cached_tokens: 0` tonen. Controleer dit met een herhaalde aanvraag met hetzelfde voorvoegsel na een korte vertraging, waarbij `usage.prompt_tokens_details.cached_tokens` als signaal voor een cachetreffer dient.

### Google Gemini (directe API)

- Direct Gemini-transport (`api: "google-generative-ai"`) rapporteert cachetreffers via upstream-`cachedContentTokenCount`, dat wordt toegewezen aan `cacheRead`.
- Geschikte modelfamilies: `gemini-2.5*` en `gemini-3*` (Live-/previewvarianten die niet met dat voorvoegsel overeenkomen, zijn uitgesloten, bijvoorbeeld `gemini-live-2.5-flash-preview`).
- Wanneer `cacheRetention` op een geschikt model is ingesteld, maakt, hergebruikt en vernieuwt OpenClaw automatisch een `cachedContents`-resource voor de systeemprompt; een handmatige cached-content-handle is niet nodig. De TTL is `300s` voor `cacheRetention: "short"` en `3600s` voor `"long"`.
- Je kunt nog steeds een bestaande Gemini-handle voor gecachte inhoud doorgeven als `params.cachedContent` (of de verouderde `params.cached_content`); bij een expliciete handle wordt het automatische cachebeheer volledig overgeslagen.
- Dit staat los van promptvoorvoegselcaching van Anthropic/OpenAI: OpenClaw beheert voor Gemini een provider-native `cachedContents`-resource in plaats van inline cachemarkeringen toe te voegen.

Bron: `src/agents/embedded-agent-runner/google-prompt-cache.ts`.

### CLI-harnasproviders (Claude Code, Gemini CLI)

CLI-backends die JSONL-gebeurtenissen voor gebruik retourneren (`jsonlDialect: "claude-stream-json"` of `"gemini-stream-json"`), worden verwerkt door een gedeelde gebruiksparser die verschillende veldnaamvarianten herkent, waaronder een gewone `cached`-teller die wordt toegewezen aan `cacheRead`. Wanneer de JSON-payload van de CLI geen rechtstreeks veld voor invoertokens bevat, leidt OpenClaw dit af als `input_tokens - cached`. Dit betreft alleen gebruiksnormalisatie; er worden geen Anthropic-/OpenAI-achtige promptcachemarkeringen gemaakt voor deze via de CLI aangestuurde modellen.

Bron: `src/agents/cli-output.ts` (`toCliUsage`).

### Andere providers

Als een provider geen van de bovenstaande cachemodi ondersteunt, heeft `cacheRetention` geen effect.

## Cachegrens van de systeemprompt

OpenClaw splitst de systeemprompt bij een interne grens voor het cachevoorvoegsel in een **stabiel voorvoegsel** en een **veranderlijk achtervoegsel**. Inhoud boven de grens (tooldefinities, metadata van Skills, werkruimtebestanden) wordt zo geordend dat deze in opeenvolgende beurten byte-identiek blijft. Inhoud onder de grens (bijvoorbeeld `HEARTBEAT.md`, runtimetijdstempels en andere metadata per beurt) kan veranderen zonder het gecachte voorvoegsel ongeldig te maken.

Belangrijkste ontwerpkeuzes:

- Stabiele projectcontextbestanden uit de werkruimte worden vóór `HEARTBEAT.md` geordend, zodat wijzigingen door Heartbeat het stabiele voorvoegsel niet verbreken.
- De grens wordt toegepast op de transportvorming voor de Anthropic-familie, OpenAI-familie, Google en CLI, zodat alle ondersteunde providers van dezelfde stabiliteit van het voorvoegsel profiteren.
- Codex Responses- en Anthropic Vertex-aanvragen worden via grensbewuste cachevorming gerouteerd, zodat cachehergebruik afgestemd blijft op wat providers daadwerkelijk ontvangen.
- Vingerafdrukken van systeemprompts worden genormaliseerd (witruimte, regeleinden, door hooks toegevoegde context en de volgorde van runtimecapaciteiten), zodat semantisch ongewijzigde prompts de cache in opeenvolgende beurten delen.

Als je na een configuratie- of werkruimtewijziging onverwachte pieken in `cacheWrite` ziet, controleer dan of de wijziging boven of onder de cachegrens terechtkomt. Door veranderlijke inhoud onder de grens te plaatsen (of te stabiliseren), wordt het probleem doorgaans opgelost.

## OpenClaw-beschermingen voor cachestabiliteit

- Gebundelde MCP-toolcatalogi worden vóór toolregistratie deterministisch gesorteerd (eerst op servernaam en daarna op toolnaam), zodat wijzigingen in de volgorde van `listTools()` het toolblok niet steeds veranderen en promptcachevoorvoegsels niet verbreken.
- Bij verouderde sessies met opgeslagen afbeeldingsblokken blijven de **3 meest recente voltooide beurten** intact (waarbij alle voltooide beurten worden geteld, niet alleen beurten met afbeeldingen). Oudere, al verwerkte afbeeldingsblokken worden vervangen door een tekstmarkering, zodat bij vervolgaanvragen met veel afbeeldingen niet steeds grote, verouderde payloads opnieuw worden verzonden.

## Afstemmingspatronen

### Gemengd verkeer (aanbevolen standaardwaarde)

Behoud een langlevende basis voor je primaire agent en schakel caching uit voor agents die in pieken meldingen verzenden:

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### Basisinstelling gericht op kosten

- Stel de basiswaarde `cacheRetention: "short"` in.
- Schakel `contextPruning.mode: "cache-ttl"` in.
- Houd Heartbeat alleen onder je TTL voor agents die voordeel hebben van warme caches.

## Live regressietests

OpenClaw voert één gecombineerde live regressiepoort voor de cache uit die herhaalde voorvoegsels, toolbeurten, afbeeldingsbeurten, MCP-achtige tooltranscripten en een Anthropic-controle zonder cache omvat.

- `src/agents/live-cache-regression.live.test.ts`
- `src/agents/live-cache-regression-runner.ts`
- `src/agents/live-cache-regression-baseline.ts`

Voer deze als volgt uit:

```sh
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache
```

Het basislijnbestand slaat de meest recent waargenomen live cijfers op, plus de providerspecifieke regressieondergrenzen waaraan de test toetst. Elke uitvoering gebruikt nieuwe sessie-ID's en promptnaamruimten per uitvoering, zodat eerdere cachestatus het huidige sample niet verstoort. Anthropic en OpenAI hanteren verschillende controles: wanneer de ondergrens bij Anthropic niet wordt gehaald, is dat een harde regressie (de test mislukt), terwijl dit bij OpenAI alleen wordt bewaakt (vastgelegd als waarschuwing; de uitvoering mislukt niet). Ze delen niet één drempelwaarde voor meerdere providers.

### Live verwachtingen voor Anthropic

- Verwacht expliciete warm-upschrijfacties via `cacheWrite`.
- Verwacht bij herhaalde beurten dat vrijwel de volledige geschiedenis wordt hergebruikt, omdat het cachebeheer van Anthropic het cachebreekpunt gedurende het gesprek verplaatst.
- De basislijnondergrenzen voor stabiele, tool-, afbeeldings- en MCP-achtige paden zijn harde regressiepoorten.

### Live verwachtingen voor OpenAI

- Verwacht alleen `cacheRead`; `cacheWrite` blijft `0` bij Chat Completions.
- Beschouw cachehergebruik bij herhaalde beurten als een providerspecifiek plateau, niet als het bewegende hergebruik van de volledige geschiedenis zoals bij Anthropic.
- Ondergrenzen dienen alleen ter bewaking (een onderschrijding wordt als waarschuwing gelogd en laat de test niet mislukken) en zijn afgeleid van waargenomen live gedrag op `gpt-5.4-mini`:

| Scenario                 | Ondergrens `cacheRead` | Ondergrens voor hitpercentage |
| ------------------------ | -----------------------------: | ----------------------------: |
| Stabiel voorvoegsel      |                          4,608 |                          0.90 |
| Tooltranscript           |                          4,096 |                          0.85 |
| Afbeeldingstranscript    |                          3,840 |                          0.82 |
| MCP-achtig transcript    |                          4,096 |                          0.85 |

De meest recent waargenomen basislijncijfers (van `live-cache-regression-baseline.ts`) kwamen uit op: stabiel voorvoegsel `cacheRead=4864`, hitpercentage `0.966`; tooltranscript `cacheRead=4608`, hitpercentage `0.896`; afbeeldingstranscript `cacheRead=4864`, hitpercentage `0.954`; MCP-achtig transcript `cacheRead=4608`, hitpercentage `0.891`.

Waarom de controles verschillen: Anthropic biedt expliciete cachebreekpunten en bewegend hergebruik van de gespreksgeschiedenis, terwijl het effectief herbruikbare voorvoegsel van OpenAI in live verkeer eerder een plateau kan bereiken dan de volledige prompt. Als beide providers aan één procentuele drempelwaarde voor meerdere providers worden getoetst, ontstaan er fout-positieve regressies.

## Configuratie voor `diagnostics.cacheTrace`

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # optioneel
    includeMessages: false # standaard true
    includePrompt: false # standaard true
    includeSystem: false # standaard true
```

Standaardwaarden:

| Sleutel           | Standaardwaarde                              |
| ----------------- | -------------------------------------------- |
| `filePath`        | `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl` |
| `includeMessages` | `true`                                       |
| `includePrompt`   | `true`                                       |
| `includeSystem`   | `true`                                       |

### Omgevingsschakelaars (eenmalige foutopsporing)

| Variabele                            | Effect                                       |
| ------------------------------------ | -------------------------------------------- |
| `OPENCLAW_CACHE_TRACE=1`             | Schakelt cachetracering in                    |
| `OPENCLAW_CACHE_TRACE_FILE=path`     | Overschrijft het uitvoerpad                   |
| `OPENCLAW_CACHE_TRACE_MESSAGES=0\|1` | Schakelt vastlegging van volledige berichten |
| `OPENCLAW_CACHE_TRACE_PROMPT=0\|1`   | Schakelt vastlegging van prompttekst          |
| `OPENCLAW_CACHE_TRACE_SYSTEM=0\|1`   | Schakelt vastlegging van de systeemprompt     |

### Wat je moet controleren

- Cachetracegebeurtenissen zijn JSONL met gefaseerde momentopnamen zoals `session:loaded`, `prompt:before`, `stream:context` en `session:after`.
- De invloed van cachetokens per beurt is zichtbaar in normale gebruiksweergaven: `cacheRead` en `cacheWrite` verschijnen in `/usage tokens`, `/status`, sessiegebruiksoverzichten en aangepaste `messages.usageTemplate`-indelingen.
- Verwacht bij Anthropic zowel `cacheRead` als `cacheWrite` wanneer caching actief is.
- Verwacht bij OpenAI `cacheRead` bij cachehits; `cacheWrite` wordt alleen ingevuld voor payloads van de Responses API die dit veld bevatten (zie [OpenAI](#openai-direct-api) hierboven).
- OpenAI retourneert ook headers voor tracering en frequentielimieten, zoals `x-request-id`, `openai-processing-ms` en `x-ratelimit-*`; gebruik deze om aanvragen te traceren, maar baseer de registratie van cachehits nog steeds op de gebruikspayload en niet op headers.

## Snelle probleemoplossing

- **Hoge `cacheWrite` bij de meeste beurten**: controleer op vluchtige invoer voor de systeemprompt; verifieer dat het model/de provider je cache-instellingen ondersteunt.
- **Hoge `cacheWrite` bij Anthropic**: betekent vaak dat het cachebreekpunt terechtkomt op inhoud die bij elke aanvraag verandert.
- **Lage OpenAI-`cacheRead`**: verifieer dat het stabiele voorvoegsel vooraan staat, het herhaalde voorvoegsel minstens 1024 tokens bevat en dezelfde `prompt_cache_key` opnieuw wordt gebruikt voor beurten die een cache moeten delen.
- **Geen effect van `cacheRetention`**: controleer of de modelsleutel overeenkomt met `agents.defaults.models["provider/model"]`.
- **Bedrock Nova-aanvragen met cache-instellingen**: verwacht — deze worden tijdens runtime omgezet naar geen cachebewaring.

Gerelateerde documentatie:

- [Anthropic](/nl/providers/anthropic)
- [Tokengebruik en kosten](/nl/reference/token-use)
- [Sessies opschonen](/nl/concepts/session-pruning)
- [Configuratiereferentie voor de Gateway](/nl/gateway/configuration-reference)

## Gerelateerd

- [Tokengebruik en kosten](/nl/reference/token-use)
- [API-gebruik en kosten](/nl/reference/api-usage-costs)
