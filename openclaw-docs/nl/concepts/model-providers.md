---
read_when:
    - Je hebt een naslagwerk nodig voor het instellen van modellen per provider
    - Je wilt voorbeeldconfiguraties of CLI-onboardingopdrachten voor modelproviders
sidebarTitle: Model providers
summary: Overzicht van modelproviders met voorbeeldconfiguraties en CLI-flows
title: Modelproviders
x-i18n:
    generated_at: "2026-07-27T05:08:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51ce1ca5dde28821596ca667619cd860cebda4787993fadb6b0e20fc0e1624ac
    source_path: concepts/model-providers.md
    workflow: 16
---

Referentie voor **LLM-/modelproviders** (niet voor chatkanalen zoals WhatsApp/Telegram). Zie [Modellen](/nl/concepts/models) voor regels voor modelselectie.

## Snelle regels

<AccordionGroup>
  <Accordion title="Modelreferenties en CLI-hulpmiddelen">
    - Modelreferenties gebruiken `provider/model` (voorbeeld: `opencode/claude-opus-4-6`).
    - `agents.defaults.models` slaat aliassen en instellingen per model op; `agents.defaults.modelPolicy.allow` is de optionele expliciete lijst met toegestane overrides.
    - CLI-hulpmiddelen: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
    - `models.providers.*.contextWindow` / `contextTokens` / `maxTokens` stellen standaardwaarden op providerniveau in; `models.providers.*.models[].contextWindow` / `contextTokens` / `maxTokens` overschrijven deze per model.
    - Regels voor terugval, cooldown-controles en persistentie van sessie-overrides: [Modelomschakeling bij fouten](/nl/concepts/model-failover).

  </Accordion>
  <Accordion title="Providerauthenticatie toevoegen wijzigt je primaire model niet">
    `openclaw configure` behoudt een bestaande `agents.defaults.model.primary` wanneer je een provider toevoegt of opnieuw authenticeert. `openclaw models auth login` doet hetzelfde, tenzij je `--set-default` meegeeft. Providerplugins kunnen nog steeds een aanbevolen standaardmodel retourneren in hun patch voor de authenticatieconfiguratie, maar OpenClaw behandelt dit als "dit model beschikbaar maken" wanneer er al een primair model bestaat, niet als "het huidige primaire model vervangen".

    Gebruik `openclaw models set <provider/model>` of `openclaw models auth login --provider <id> --set-default` om bewust van standaardmodel te wisselen.

  </Accordion>
  <Accordion title="Scheiding tussen OpenAI-provider en -runtime">
    OpenAI-modelreferenties en agentruntimes staan los van elkaar:

    - `openai/<model>` selecteert de canonieke OpenAI-provider en het model. Alleen het voorvoegsel selecteert nooit Codex.
    - Als het runtimebeleid voor provider/model niet is ingesteld of `auto` is, mag OpenAI Codex alleen impliciet selecteren voor een exacte officiële HTTPS-route voor Platform Responses of ChatGPT Responses zonder zelf opgegeven requestoverride.
    - Zelf opgegeven Completions-adapters, aangepaste eindpunten en routes met zelf opgegeven requestgedrag blijven op OpenClaw. Officiële HTTP-eindpunten met platte tekst worden geweigerd.
    - Verouderde Codex-modelreferenties zijn verouderde configuratie die doctor herschrijft naar `openai/<model>`.
    - Provider/model `agentRuntime.id: "openclaw"` houdt een anderszins geschikte route expliciet op OpenClaw. `agentRuntime.id: "codex"` vereist Codex en stopt veilig met een fout wanneer de effectieve route niet compatibel is met Codex.

    Zie [Impliciete OpenAI-agentruntime](/nl/providers/openai#implicit-agent-runtime) en [Codex-harnas](/nl/plugins/codex-harness). Als de scheiding tussen provider en runtime verwarrend is, lees dan eerst [Agentruntimes](/nl/concepts/agent-runtimes).

    Het automatisch inschakelen van plugins volgt dezelfde grens: een impliciet met Codex compatibele effectieve route kan de Codex-plugin inschakelen, terwijl expliciete provider/model-`agentRuntime.id: "codex"`- of verouderde `codex/<model>`-referenties deze vereisen. Alleen een `openai/*`-voorvoegsel doet dat niet.

    Een nieuwe OpenAI-configuratie gebruikt een routespecifieke GPT-5.6-referentie: configuratie met een API-sleutel selecteert
    `openai/gpt-5.6` (de kale directe-API-id wordt omgezet naar Sol), terwijl
    ChatGPT/Codex OAuth exact `openai/gpt-5.6-sol` selecteert voor de systeemeigen Codex-
    catalogus. Bestaande expliciete primaire modellen, waaronder `openai/gpt-5.5`, blijven
    behouden wanneer OpenAI-authenticatie wordt toegevoegd of vernieuwd. GPT-5.5 blijft
    via beide runtimes beschikbaar als expliciete herstelkeuze voor accounts zonder
    toegang tot GPT-5.6.

  </Accordion>
  <Accordion title="CLI-runtimes">
    CLI-runtimes gebruiken dezelfde scheiding: kies canonieke modelreferenties zoals `anthropic/claude-*` of `google/gemini-*` en stel vervolgens het runtimebeleid voor provider/model in op `claude-cli` of `google-gemini-cli` wanneer je een lokale CLI-backend wilt.

    Verouderde `claude-cli/*`- en `google-gemini-cli/*`-referenties worden terug gemigreerd naar canonieke providerreferenties, waarbij de runtime afzonderlijk wordt vastgelegd. Verouderde `codex-cli/*`-referenties worden gemigreerd naar `openai/*` en gebruiken de Codex-app-serverroute; OpenClaw bevat niet langer een gebundelde Codex CLI-backend.

  </Accordion>
</AccordionGroup>

## Providers configureren in de Control UI

Open **Settings → Model Providers** in de Control UI om provider-API-sleutels die zijn opgeslagen in `models.providers.<id>.apiKey` toe te voegen, te vervangen of te verwijderen. De pagina geeft aan of elke API-sleutel afkomstig is uit de OpenClaw-configuratie of uit een omgevingsvariabele, zonder de referentie te tonen. Sleutels uit de omgeving blijven beheerd door de procesomgeving van de Gateway.

Gebruik **Test connection** om een live providercontrole uit te voeren en de latentie of een gecategoriseerde authenticatie-, snelheidslimiet-, facturerings-, time-out- of responsfout te bekijken. Een controle voert een echt providerrequest uit en kan een klein aantal tokens verbruiken. Je kunt OAuth- en tokenprofielen ook afmelden via de providerkaart.

De kaart **Default models** beheert het primaire model, de geordende terugvalmodellen en het hulpmiddelmodel uit de geconfigureerde modelcatalogus. Kies de modellen en sla ze vervolgens samen op in de bestaande instellingen `agents.defaults.model` en `agents.defaults.utilityModel`. Voor het hulpmiddelmodel laat **Automatic** de instelling oningesteld en slaat **Disabled** een lege tekenreeks op om routering naar hulpmiddelen uit te schakelen.

## Providergedrag dat eigendom is van plugins

De meeste providerspecifieke logica bevindt zich in providerplugins (`registerProvider(...)`), terwijl OpenClaw de generieke inferentielus beheert. Plugins zijn verantwoordelijk voor onboarding, modelcatalogi, toewijzing van authenticatieomgevingsvariabelen, normalisatie van transport/configuratie, opschoning van toolschema's, classificatie voor omschakeling bij fouten, vernieuwing van OAuth, gebruiksrapportage, denk-/redeneerprofielen en meer.

De volledige lijst met provider-SDK-hooks en voorbeelden van gebundelde plugins staat in [Providerplugins](/nl/plugins/sdk-provider-plugins). Een provider die een volledig aangepaste requestexecutor nodig heeft, is een afzonderlijk, dieper uitbreidingsoppervlak.

<Note>
Providergestuurd runnergedrag bevindt zich in expliciete providerhooks, zoals beleid voor opnieuw afspelen, normalisatie van toolschema's, streamwrapping en transport-/requesthulpmiddelen. De verouderde statische verzameling `ProviderPlugin.capabilities` is alleen bedoeld voor compatibiliteit en wordt niet langer gelezen door gedeelde runnerlogica.
</Note>

## Rotatie van API-sleutels

<AccordionGroup>
  <Accordion title="Sleutelbronnen en prioriteit">
    Configureer meerdere sleutels via:

    - `OPENCLAW_LIVE_<PROVIDER>_KEY` (enkele live override, hoogste prioriteit)
    - `<PROVIDER>_API_KEYS` (lijst gescheiden door komma's of puntkomma's)
    - `<PROVIDER>_API_KEY` (primaire sleutel)
    - `<PROVIDER>_API_KEY_*` (genummerde lijst, bijvoorbeeld `<PROVIDER>_API_KEY_1`)

    Voor Google-providers wordt `GOOGLE_API_KEY` ook als terugvaloptie opgenomen. De selectievolgorde voor sleutels behoudt de prioriteit en verwijdert dubbele waarden.

  </Accordion>
  <Accordion title="Wanneer rotatie wordt geactiveerd">
    - Requests worden alleen opnieuw geprobeerd met de volgende sleutel bij reacties over snelheidslimieten (bijvoorbeeld `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded` of periodieke berichten over gebruikslimieten).
    - Fouten die niet door snelheidslimieten worden veroorzaakt, mislukken direct; er wordt geen sleutelrotatie geprobeerd.
    - Wanneer alle kandidaat-sleutels mislukken, wordt de uiteindelijke fout van de laatste poging geretourneerd.

  </Accordion>
</AccordionGroup>

## Officiële providerplugins

Officiële providerplugins publiceren hun eigen modelcatalogusrijen. Deze providers vereisen **geen** `models.providers`-modelvermeldingen; schakel de providerplugin in, stel de authenticatie in en kies een model. Gebruik `models.providers` alleen voor expliciete aangepaste providers of specifieke requestinstellingen zoals time-outs.

### OpenAI

- Provider: `openai`
- Authenticatie: `OPENAI_API_KEY`
- Optionele rotatie: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, plus `OPENCLAW_LIVE_OPENAI_KEY` (enkele override)
- Standaard voor nieuwe configuraties: `openai/gpt-5.6`; op de directe API wordt de kale id omgezet naar Sol.
- Voorbeeldmodellen: `openai/gpt-5.6`, `openai/gpt-5.6-terra`, `openai/gpt-5.6-luna`, `openai/gpt-5.5`
- Controleer de beschikbaarheid voor account/model met `openclaw models list --provider openai` als een specifieke installatie of API-sleutel zich anders gedraagt.
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Het standaardtransport is `auto`; OpenClaw geeft de transportkeuze door aan de gedeelde modelruntime.
- Overschrijf dit per model via `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` of `"auto"`)
- Prioriteitsverwerking van OpenAI kan worden ingeschakeld via `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` en `params.fastMode` wijzen directe `openai/*` Responses-requests toe aan `service_tier=priority` op `api.openai.com`
- Gebruik `params.serviceTier` wanneer je een expliciet niveau wilt in plaats van de gedeelde schakelaar `/fast`
- Verborgen OpenClaw-toeschrijvingsheaders (`originator`, `version`, `User-Agent`) zijn alleen van toepassing op systeemeigen OpenAI-verkeer naar `api.openai.com`, niet op generieke OpenAI-compatibele proxy's
- Systeemeigen OpenAI-routes behouden ook Responses-`store`, hints voor de promptcache en vormgeving van payloads voor OpenAI-redeneercompatibiliteit; proxyroutes doen dat niet
- `openai/gpt-5.3-codex-spark` is alleen beschikbaar via ChatGPT/Codex OAuth; routes met een directe OpenAI-API-sleutel en Azure-API-sleutel weigeren dit

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
}
```

Als de API-organisatie GPT-5.6 niet beschikbaar stelt, stel je
`openai/gpt-5.5` expliciet in. Normale onboarding en herauthenticatie behouden een
bestaand expliciet primair model; `models auth login --set-default` en
`models set` zijn de bewuste vervangingspaden.

### Anthropic

- Provider: `anthropic`
- Authenticatie: `ANTHROPIC_API_KEY`
- Optionele rotatie: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, plus `OPENCLAW_LIVE_ANTHROPIC_KEY` (enkele override)
- Voorbeeldmodel: `anthropic/claude-opus-5`
- CLI: `openclaw onboard --auth-choice apiKey`
- Directe openbare Anthropic-requests ondersteunen de gedeelde schakelaar `/fast` en `params.fastMode`, waaronder verkeer dat met een API-sleutel of OAuth-authenticatie naar `api.anthropic.com` wordt verzonden; OpenClaw wijst dit toe aan Anthropic `service_tier` (`auto` versus `standard_only`)
- De voorkeursconfiguratie voor Claude CLI houdt de modelreferentie canoniek en selecteert de CLI-
  backend afzonderlijk: `anthropic/claude-opus-5` met
  modelgebonden `agentRuntime.id: "claude-cli"`. Verouderde
  `claude-cli/claude-opus-4-7`-referenties blijven werken voor compatibiliteit.

<Note>
Hergebruik van Claude CLI (`claude -p`) is een goedgekeurd OpenClaw-integratiepad. Authenticatie met een Anthropic-installatietoken blijft ondersteund, maar OpenClaw geeft waar mogelijk de voorkeur aan hergebruik van Claude CLI.
</Note>

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
}
```

### OpenAI ChatGPT/Codex OAuth

- Provider: `openai`
- Authenticatie: OAuth (ChatGPT)
- Referentie voor verse systeemeigen Codex-app-serverharness: `openai/gpt-5.6-sol`
- Documentatie voor systeemeigen Codex-app-serverharness: [Codex-harness](/nl/plugins/codex-harness)
- Verouderde modelreferenties: `codex/gpt-*`, `openai-codex/gpt-*`
- Plugin-grens: `openai/*` laadt de OpenAI-plugin; expliciet runtimebeleid of de effectieve route van de provider bepaalt of de systeemeigen Codex-app-serverplugin wordt geselecteerd.
- CLI: `openclaw onboard --auth-choice openai` of `openclaw models auth login --provider openai`
- Het ingebedde ChatGPT Responses-transport van OpenClaw gebruikt standaard `auto` (eerst WebSocket, met SSE als fallback).
- `agents.defaults.models["openai/<model>"].params.transport`, `params.serviceTier` en `params.fastMode` zijn ingestelde instellingen voor ingebedde aanvragen. Daarmee blijft de impliciete runtimeselectie bij OpenClaw; systeemeigen Codex beheert het app-servertransport en serviceniveau.
- Verborgen OpenClaw-attributieheaders (`originator`, `version`, `User-Agent`) worden alleen toegevoegd aan systeemeigen Codex-verkeer naar `chatgpt.com/backend-api`, niet aan algemene OpenAI-compatibele proxy's
- De gedeelde schakeloptie `/fast` blijft beschikbaar als runtimebesturing; deze staat los van ingestelde modelparameters.
- De systeemeigen Codex-catalogus kan afhankelijk van de accounttoegang exacte referenties voor `openai/gpt-5.6-sol`, `openai/gpt-5.6-terra` en `openai/gpt-5.6-luna` beschikbaar stellen. De kale alias `gpt-5.6` van de directe API wordt niet aan de clientzijde toegepast.
- `openai/gpt-5.5` gebruikt de systeemeigen Codex-catalogus `contextWindow = 400000` en de standaardruntime `contextTokens = 272000`; overschrijf de runtimelimiet met `models.providers.openai.models[].contextTokens`
- Meld je aan met `openai`-authenticatie en gebruik `openai/gpt-5.6-sol` voor een nieuwe, door een abonnement ondersteunde configuratie. Selecteer `openai/gpt-5.5` expliciet als GPT-5.6 niet beschikbaar is in die Codex-werkruimte.
- Gebruik provider/model `agentRuntime.id: "openclaw"` om een verder geschikte route op de ingebouwde runtime te houden. Als de runtime niet is ingesteld of `auto` is, kan Codex alleen impliciet worden geselecteerd voor een exacte officiële HTTPS-route die compatibel is met Responses/ChatGPT en geen ingestelde aanvraagoverschrijving heeft.
- Verouderde Codex GPT-referenties zijn verouderde status, geen actieve providerroute. Gebruik canonieke `openai/*`-referenties voor nieuwe agentconfiguratie en voer `openclaw doctor --fix` uit om `codex/*`- en `openai-codex/*`-referenties te migreren, waarbij hun systeemeigen Codex-semantiek behouden blijft met modelgebonden `agentRuntime.id: "codex"`. Bestaande expliciete canonieke `openai/gpt-5.5`-selecties worden niet bijgewerkt.

```json5
{
  plugins: { entries: { codex: { enabled: true } } },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
    },
  },
}
```

```json5
{
  models: {
    providers: {
      openai: {
        models: [{ id: "gpt-5.5", contextTokens: 160000 }],
      },
    },
  },
}
```

### Andere gehoste opties in abonnementsstijl

<CardGroup cols={3}>
  <Card title="MiniMax" href="/nl/providers/minimax">
    Toegang via MiniMax Coding Plan OAuth of API-sleutel.
  </Card>
  <Card title="Qwen Cloud" href="/nl/providers/qwen">
    Provideroppervlak van Qwen Cloud plus eindpunttoewijzing voor Alibaba DashScope en Coding Plan.
  </Card>
  <Card title="Z.AI (GLM)" href="/nl/providers/zai">
    Z.AI Coding Plan of algemene API-eindpunten.
  </Card>
</CardGroup>

### OpenCode

- Authenticatie: `OPENCODE_API_KEY` (of `OPENCODE_ZEN_API_KEY`)
- Zen-runtimeprovider: `opencode`
- Go-runtimeprovider: `opencode-go`
- Voorbeeldmodellen: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice opencode-zen` of `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (API-sleutel)

- Provider: `google`
- Authenticatie: `GEMINI_API_KEY`
- Optionele rotatie: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, `GOOGLE_API_KEY` als fallback en `OPENCLAW_LIVE_GEMINI_KEY` (één overschrijving)
- Voorbeeldmodellen: `google/gemini-3.1-pro-preview`, `google/gemini-3.5-flash`
- Compatibiliteit: verouderde OpenClaw-configuratie met `google/gemini-3.1-flash-preview` wordt genormaliseerd naar `google/gemini-3-flash-preview`
- Alias: `google/gemini-3.1-pro` wordt geaccepteerd en genormaliseerd naar de actieve Gemini API-id van Google, `google/gemini-3.1-pro-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Redeneren: `/think adaptive` gebruikt dynamisch redeneren van Google. Gemini 3/3.1 laten een vaste `thinkingLevel` weg; Gemini 2.5 verzendt `thinkingBudget: -1`.
- Directe Gemini-runs accepteren ook `agents.defaults.models["google/<model>"].params.cachedContent` (of het verouderde `cached_content`) om een providerspecifieke `cachedContents/...`-handle door te sturen; Gemini-cachetreffers worden weergegeven als OpenClaw `cacheRead`

### Google Vertex en Gemini CLI

- Providers: `google-vertex`, `google-gemini-cli`
- Authenticatie: Vertex gebruikt gcloud ADC; Gemini CLI gebruikt de eigen OAuth-flow

<Warning>
Gemini CLI OAuth in OpenClaw is een onofficiële integratie. Sommige gebruikers hebben gemeld dat hun Google-account werd beperkt na het gebruik van clients van derden. Lees de voorwaarden van Google en gebruik een niet-kritiek account als je besluit door te gaan.
</Warning>

Gemini CLI OAuth wordt geleverd als onderdeel van de gebundelde `google`-plugin.

<Steps>
  <Step title="Gemini CLI installeren">
    <Tabs>
      <Tab title="brew">
        ```bash
        brew install gemini-cli
        ```
      </Tab>
      <Tab title="npm">
        ```bash
        npm install -g @google/gemini-cli
        ```
      </Tab>
    </Tabs>
  </Step>
  <Step title="Plugin inschakelen">
    ```bash
    openclaw plugins enable google
    ```
  </Step>
  <Step title="Aanmelden">
    ```bash
    openclaw models auth login --provider google-gemini-cli --set-default
    ```

    Standaardmodel: `google-gemini-cli/gemini-3-flash-preview`. Je plakt **geen** client-id of geheim in `openclaw.json`. De CLI-aanmeldingsflow slaat tokens op in authenticatieprofielen op de Gateway-host.

  </Step>
  <Step title="Project instellen (indien nodig)">
    Als aanvragen na het aanmelden mislukken, stel je `GOOGLE_CLOUD_PROJECT` of `GOOGLE_CLOUD_PROJECT_ID` in op de Gateway-host.
  </Step>
</Steps>

Gemini CLI gebruikt standaard `stream-json`. OpenClaw leest streamberichten
van de assistent en normaliseert `stats.cached` naar `cacheRead`; verouderde
`--output-format json`-overschrijvingen lezen antwoordtekst nog steeds uit `response`.

### Z.AI (GLM)

- Provider: `zai`
- Authenticatie: `ZAI_API_KEY`
- Voorbeeldmodel: `zai/glm-5.2`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Modelreferenties gebruiken de canonieke provider-id `zai/*`.
  - `zai-api-key` detecteert automatisch het bijbehorende Z.AI-eindpunt; `zai-coding-global`, `zai-coding-cn`, `zai-global` en `zai-cn` dwingen een specifiek oppervlak af

### Vercel AI Gateway

- Provider: `vercel-ai-gateway`
- Authenticatie: `AI_GATEWAY_API_KEY`
- Voorbeeldmodellen: `vercel-ai-gateway/anthropic/claude-opus-4.6`, `vercel-ai-gateway/moonshotai/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Andere gebundelde providerplugins

| Provider                                | Id                               | Auth-omgeving                                        | Voorbeeldmodel                                         |
| --------------------------------------- | -------------------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| Arcee                                   | `arcee`                          | `ARCEEAI_API_KEY` of `OPENROUTER_API_KEY`            | `arcee/trinity-large-thinking`                         |
| BytePlus                                | `byteplus` / `byteplus-plan`     | `BYTEPLUS_API_KEY`                                   | `byteplus-plan/ark-code-latest`                        |
| Cerebras                                | `cerebras`                       | `CEREBRAS_API_KEY`                                   | `cerebras/zai-glm-4.7`                                 |
| Chutes                                  | `chutes`                         | `CHUTES_API_KEY` of `CHUTES_OAUTH_TOKEN`             | `chutes/zai-org/GLM-5-TEE`                             |
| ClawRouter                              | `clawrouter`                     | `CLAWROUTER_API_KEY`                                 | `clawrouter/anthropic/claude-sonnet-4-6`               |
| Cohere                                  | `cohere`                         | `COHERE_API_KEY`                                     | `cohere/command-a-plus-05-2026`                        |
| DeepInfra                               | `deepinfra`                      | `DEEPINFRA_API_KEY`                                  | `deepinfra/deepseek-ai/DeepSeek-V4-Flash`              |
| DeepSeek                                | `deepseek`                       | `DEEPSEEK_API_KEY`                                   | `deepseek/deepseek-v4-flash`                           |
| Featherless AI                          | `featherless`                    | `FEATHERLESS_API_KEY`                                | `featherless/Qwen/Qwen3-32B`                           |
| GitHub Copilot                          | `github-copilot`                 | `COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN` | -                                                      |
| GMI Cloud                               | `gmi`                            | `GMI_API_KEY`                                        | `gmi/google/gemini-3.1-flash-lite`                     |
| Groq                                    | `groq`                           | `GROQ_API_KEY`                                       | `groq/llama-3.3-70b-versatile`                         |
| Hugging Face Inference                  | `huggingface`                    | `HUGGINGFACE_HUB_TOKEN` of `HF_TOKEN`                | `huggingface/deepseek-ai/DeepSeek-R1`                  |
| MiniMax                                 | `minimax` / `minimax-portal`     | `MINIMAX_API_KEY` / `MINIMAX_OAUTH_TOKEN`            | `minimax/MiniMax-M3`                                   |
| Mistral                                 | `mistral`                        | `MISTRAL_API_KEY`                                    | `mistral/mistral-large-latest`                         |
| Moonshot                                | `moonshot`                       | `MOONSHOT_API_KEY`                                   | `moonshot/kimi-k2.6`                                   |
| NVIDIA                                  | `nvidia`                         | `NVIDIA_API_KEY`                                     | `nvidia/nvidia/nemotron-3-ultra-550b-a55b`             |
| NovitaAI                                | `novita`                         | `NOVITA_API_KEY`                                     | `novita/deepseek/deepseek-v3-0324`                     |
| [Ollama Cloud](/nl/providers/ollama-cloud) | `ollama-cloud`                   | `OLLAMA_API_KEY`                                     | `ollama-cloud/kimi-k2.6`                               |
| OpenRouter                              | `openrouter`                     | OpenRouter OAuth of `OPENROUTER_API_KEY`             | `openrouter/auto`                                      |
| Qianfan                                 | `qianfan`                        | `QIANFAN_API_KEY`                                    | `qianfan/deepseek-v3.2`                                |
| Tencent TokenHub                        | `tencent-tokenhub`               | `TOKENHUB_API_KEY`                                   | `tencent-tokenhub/hy3-preview`                         |
| Together                                | `together`                       | `TOGETHER_API_KEY`                                   | `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`     |
| Venice                                  | `venice`                         | `VENICE_API_KEY`                                     | -                                                      |
| Vercel AI Gateway                       | `vercel-ai-gateway`              | `AI_GATEWAY_API_KEY`                                 | `vercel-ai-gateway/anthropic/claude-opus-4.6`          |
| Volcano Engine (Doubao)                 | `volcengine` / `volcengine-plan` | `VOLCANO_ENGINE_API_KEY`                             | `volcengine-plan/ark-code-latest`                      |
| xAI                                     | `xai`                            | SuperGrok/X Premium OAuth of `XAI_API_KEY`           | `xai/grok-4.3`                                         |
| Xiaomi                                  | `xiaomi` / `xiaomi-token-plan`   | `XIAOMI_API_KEY` / `XIAOMI_TOKEN_PLAN_API_KEY`       | `xiaomi/mimo-v2.5` / `xiaomi-token-plan/mimo-v2.5-pro` |

#### Bijzonderheden die je moet kennen

<AccordionGroup>
  <Accordion title="OpenRouter">
    Past de headers voor app-toeschrijving en Anthropic-markeringen voor `cache_control` alleen toe op geverifieerde `openrouter.ai`-routes. Verwijzingen naar DeepSeek, Moonshot en ZAI komen in aanmerking voor cache-TTL bij door OpenRouter beheerde promptcaching, maar ontvangen geen Anthropic-cachemarkeringen. Als proxyachtige OpenAI-compatibele route slaat deze uitsluitend voor native OpenAI bedoelde vormgeving over (`serviceTier`, Responses `store`, hints voor de promptcache, compatibiliteit met OpenAI-redenering). Verwijzingen met Gemini als backend behouden alleen de opschoning van proxy-Gemini-denksignaturen.
  </Accordion>
  <Accordion title="Kilo Gateway">
    Verwijzingen met Gemini als backend volgen hetzelfde opschoningspad voor proxy-Gemini; `kilocode/kilo-auto/balanced` en andere verwijzingen die proxyredenering niet ondersteunen, slaan de injectie van proxyredenering over.
  </Accordion>
  <Accordion title="MiniMax">
    Onboarding met een API-sleutel schrijft expliciete M3- en M2.7-chatmodeldefinities; beeldherkenning blijft bij de door de plugin beheerde mediaprovider `MiniMax-VL-01`.
  </Accordion>
  <Accordion title="NVIDIA">
    Model-id's gebruiken een `nvidia/<vendor>/<model>`-naamruimte (bijvoorbeeld `nvidia/nvidia/nemotron-...`); keuzelijsten behouden de letterlijke `<provider>/<model-id>`-samenstelling, terwijl de canonieke sleutel die naar de API wordt verzonden één voorvoegsel behoudt.
  </Accordion>
  <Accordion title="xAI">
    Gebruikt het xAI Responses-pad. Het aanbevolen pad is SuperGrok/X Premium OAuth; API-sleutels werken nog steeds via `XAI_API_KEY` of pluginconfiguratie, en Grok `web_search` gebruikt vóór de terugval op een API-sleutel hetzelfde authenticatieprofiel opnieuw. Grok 4.5 kan, waar beschikbaar, worden geselecteerd voor chatten, programmeren en agentgestuurd werk; `grok-4.3` blijft de gebundelde standaard die veilig is voor alle regio's. Oudere configuraties met `/fast` en `params.fastMode: true` worden nog steeds afgehandeld via de Grok 4.3-compatibiliteitsomleidingen van xAI, maar nieuwe configuraties moeten rechtstreeks een actueel model selecteren. `tool_stream` is standaard ingeschakeld; schakel dit uit via `agents.defaults.models["xai/<model>"].params.tool_stream=false`.
  </Accordion>
</AccordionGroup>

## Providers via `models.providers` (aangepaste/basis-URL)

Gebruik `models.providers` (of `models.json`) om **aangepaste** providers of OpenAI-/Anthropic-compatibele proxy's toe te voegen.

Veel van de onderstaande gebundelde providerplugins publiceren al een standaardcatalogus. Gebruik expliciete `models.providers.<id>`-vermeldingen alleen als je de standaardbasis-URL, headers of modellenlijst wilt overschrijven.

Gebundelde en in de catalogus bekende routes ontlenen hun `compat`-mogelijkheden aan de verantwoordelijke providerplugin. Een configuratieblok `compat` is bedoeld voor een aangepaste provider of een aangepast model, of voor een andere `api`-/`baseUrl`-route waarvan je het endpointcontract hebt geverifieerd; zie de [handleiding voor mogelijkheden van aangepaste providers](/nl/gateway/config-tools#custom-provider-capability-declarations). Doctor verwijdert verouderde waarden die slechts de catalogus herhalen en laat afwijkende waarden zichtbaar voor beoordeling door de beheerder.

Controles van modelmogelijkheden door de Gateway lezen ook expliciete `models.providers.<id>.models[]`-metadata. Als een aangepast model of proxymodel afbeeldingen accepteert, stel je `input: ["text", "image"]` in voor dat model, zodat WebChat en bijlagenpaden die vanuit een Node afkomstig zijn afbeeldingen doorgeven als native modelinvoer in plaats van alleen-tekstmediaverwijzingen.

`agents.defaults.models["provider/model"]` beheert aliassen en metadata per model voor agents. Het beperkt geen overschrijvingen en registreert op zichzelf ook geen nieuw runtimemodel. Voeg voor modellen van aangepaste providers ook `models.providers.<provider>.models[]` toe met ten minste de overeenkomende `id`; gebruik `agents.defaults.modelPolicy.allow` afzonderlijk wanneer je een beperking voor overschrijvingen wilt.

### Moonshot AI (Kimi)

Installeer `@openclaw/moonshot-provider` vóór de onboarding. Voeg alleen een expliciete `models.providers.moonshot`-vermelding toe als je de basis-URL of modelmetadata moet overschrijven:

- Provider: `moonshot`
- Authenticatie: `MOONSHOT_API_KEY`
- Voorbeeldmodel: `moonshot/kimi-k3`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` of `openclaw onboard --auth-choice moonshot-api-key-cn`

Kimi-model-id's:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.6`
- `moonshot/kimi-k3`
- `moonshot/kimi-k2.7-code`
- `moonshot/kimi-k2.7-code-highspeed`
- `moonshot/kimi-k2.5`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.6" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.6", name: "Kimi K2.6" }],
      },
    },
  },
}
```

Zie [Moonshot AI (Kimi + Kimi Coding)](/nl/providers/moonshot) voor de volledige installatiehandleiding.

### Kimi Coding

Kimi Coding gebruikt het Anthropic-compatibele endpoint van Moonshot AI:

- Provider: `kimi`
- Authenticatie: `KIMI_API_KEY`
- Kimi K3: `kimi/k3` (256K) of `kimi/k3[1m]` (1M-abonnement)
- Kimi Code: `kimi/kimi-for-coding`
- Kimi Code HighSpeed: `kimi/kimi-for-coding-highspeed`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-for-coding" } },
  },
}
```

Verouderde `kimi/kimi-code` en `kimi/k2p5` blijven geaccepteerd als compatibiliteitsmodel-id's en worden genormaliseerd naar het stabiele API-model-id van Kimi.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) biedt in China toegang tot Doubao en andere modellen.

- Provider: `volcengine` (programmeren: `volcengine-plan`)
- Authenticatie: `VOLCANO_ENGINE_API_KEY`
- Voorbeeldmodel: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

Onboarding gebruikt standaard het programmeeroppervlak, maar de algemene `volcengine/*`-catalogus wordt tegelijkertijd geregistreerd.

In de modelkeuzelijsten voor onboarding/configuratie geeft de Volcengine-authenticatiekeuze de voorkeur aan zowel `volcengine/*`- als `volcengine-plan/*`-rijen. Als die modellen nog niet zijn geladen, valt OpenClaw terug op de ongefilterde catalogus in plaats van een lege, tot de provider beperkte keuzelijst weer te geven.

<Tabs>
  <Tab title="Standaardmodellen">
    - `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
    - `volcengine/doubao-seed-code-preview-251028`
    - `volcengine/kimi-k2-5-260127` (Kimi K2.5)
    - `volcengine/glm-4-7-251222` (GLM 4.7)
    - `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2)

  </Tab>
  <Tab title="Codeermodellen (volcengine-plan)">
    - `volcengine-plan/ark-code-latest`
    - `volcengine-plan/doubao-seed-code`

  </Tab>
</Tabs>

### BytePlus (internationaal)

BytePlus ARK biedt internationale gebruikers toegang tot dezelfde modellen als Volcano Engine.

- Provider: `byteplus` (coderen: `byteplus-plan`)
- Authenticatie: `BYTEPLUS_API_KEY`
- Voorbeeldmodel: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

De onboarding gebruikt standaard het codeeroppervlak, maar de algemene `byteplus/*`-catalogus wordt tegelijkertijd geregistreerd.

In modelkiezers voor onboarding/configuratie geeft de BytePlus-authenticatiekeuze de voorkeur aan zowel de rijen `byteplus/*` als `byteplus-plan/*`. Als die modellen nog niet zijn geladen, valt OpenClaw terug op de ongefilterde catalogus in plaats van een lege, tot de provider beperkte kiezer te tonen.

<Tabs>
  <Tab title="Standaardmodellen">
    - `byteplus/seed-1-8-251228` (Seed 1.8)
    - `byteplus/kimi-k2-5-260127` (Kimi K2.5)
    - `byteplus/glm-4-7-251222` (GLM 4.7)

  </Tab>
  <Tab title="Codeermodellen (byteplus-plan)">
    - `byteplus-plan/ark-code-latest`
    - `byteplus-plan/kimi-k2.5`
    - `byteplus-plan/glm-4.7`

  </Tab>
</Tabs>

### Synthetic

Synthetic biedt Anthropic-compatibele modellen via de provider `synthetic`:

- Provider: `synthetic`
- Authenticatie: `SYNTHETIC_API_KEY`
- Voorbeeldmodel: `synthetic/hf:MiniMaxAI/MiniMax-M3`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M3", name: "MiniMax M3" }],
      },
    },
  },
}
```

### MiniMax

MiniMax wordt via `models.providers` geconfigureerd omdat het aangepaste eindpunten gebruikt:

- MiniMax OAuth (wereldwijd): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax-API-sleutel (wereldwijd): `--auth-choice minimax-global-api`
- MiniMax-API-sleutel (CN): `--auth-choice minimax-cn-api`
- Authenticatie: `MINIMAX_API_KEY` voor `minimax`; `MINIMAX_OAUTH_TOKEN` of `MINIMAX_API_KEY` voor `minimax-portal`

Zie [/providers/minimax](/nl/providers/minimax) voor installatiedetails, modelopties en configuratiefragmenten.

<Note>
Op het Anthropic-compatibele streamingpad van MiniMax schakelt OpenClaw denken standaard uit voor de M2.x-familie, tenzij je dit expliciet instelt; MiniMax-M3 (en M3.x) blijft standaard op het weggelaten/adaptieve denkpad van de provider. `/fast on` herschrijft `MiniMax-M2.7` naar `MiniMax-M2.7-highspeed`.
</Note>

Door de Plugin beheerde verdeling van mogelijkheden:

- Standaardinstellingen voor tekst/chat blijven op `minimax/MiniMax-M3`
- Afbeeldingen genereren gebruikt `minimax/image-01` of `minimax-portal/image-01`
- Afbeeldingsbegrip gebruikt de door de Plugin beheerde `MiniMax-VL-01` op beide MiniMax-authenticatiepaden
- Zoeken op internet blijft op provider-id `minimax`

### LM Studio

LM Studio wordt geleverd als een gebundelde provider-Plugin die de systeemeigen API gebruikt:

- Provider: `lmstudio`
- Authenticatie: `LM_API_TOKEN`
- Standaardbasis-URL voor inferentie: `http://localhost:1234/v1`

Stel vervolgens een model in (vervang dit door een van de ID's die `http://localhost:1234/api/v1/models` retourneert):

```json5
{
  agents: {
    defaults: { model: { primary: "lmstudio/openai/gpt-oss-20b" } },
  },
}
```

OpenClaw gebruikt de systeemeigen `/api/v1/models` en `/api/v1/models/load` van LM Studio voor detectie en automatisch laden, met standaard `/v1/chat/completions` voor inferentie. Als je wilt dat JIT-laden, TTL en automatisch verwijderen van LM Studio de modellevenscyclus beheren, stel je `models.providers.lmstudio.params.preload: false` in. Zie [/providers/lmstudio](/nl/providers/lmstudio) voor installatie en probleemoplossing.

### Ollama

Ollama wordt geleverd als een gebundelde provider-Plugin en gebruikt de systeemeigen API van Ollama:

- Provider: `ollama`
- Authenticatie: Niet vereist (lokale server)
- Voorbeeldmodel: `ollama/llama3.3`
- Installatie: [https://ollama.com/download](https://ollama.com/download)

```bash
# Installeer Ollama en haal vervolgens een model op:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama wordt lokaal gedetecteerd op `http://127.0.0.1:11434` wanneer je dit inschakelt met `OLLAMA_API_KEY`, en de gebundelde provider-Plugin voegt Ollama rechtstreeks toe aan `openclaw onboard` en de modelkiezer. Zie [/providers/ollama](/nl/providers/ollama) voor onboarding, cloud-/lokale modus en aangepaste configuratie.

### vLLM

vLLM wordt geleverd als een gebundelde provider-Plugin voor lokale/zelfgehoste OpenAI-compatibele servers:

- Provider: `vllm`
- Authenticatie: Optioneel (afhankelijk van je server)
- Standaardbasis-URL: `http://127.0.0.1:8000/v1`

Om lokale automatische detectie in te schakelen (elke waarde werkt als je server geen authenticatie afdwingt):

```bash
export VLLM_API_KEY="vllm-local"
```

Stel vervolgens een model in (vervang dit door een van de ID's die `/v1/models` retourneert):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Zie [/providers/vllm](/nl/providers/vllm) voor details.

### SGLang

SGLang wordt geleverd als een gebundelde provider-Plugin voor snelle, zelfgehoste OpenAI-compatibele servers:

- Provider: `sglang`
- Authenticatie: Optioneel (afhankelijk van je server)
- Standaardbasis-URL: `http://127.0.0.1:30000/v1`

Om lokale automatische detectie in te schakelen (elke waarde werkt als je server geen authenticatie afdwingt):

```bash
export SGLANG_API_KEY="sglang-local"
```

Stel vervolgens een model in (vervang dit door een van de ID's die `/v1/models` retourneert):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Zie [/providers/sglang](/nl/providers/sglang) voor details.

### Lokale proxy's (LM Studio, vLLM, LiteLLM enz.)

Voorbeeld (OpenAI-compatibel):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "${LM_API_TOKEN}",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Standaard optionele velden">
    Voor aangepaste providers zijn `reasoning`, `input`, `cost`, `contextWindow` en `maxTokens` optioneel. Als ze worden weggelaten, gebruikt OpenClaw standaard:

    - `reasoning: false`
    - `input: ["text"]`
    - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
    - `contextWindow: 200000`
    - `maxTokens: 8192`

    Aanbevolen: stel expliciete waarden in die overeenkomen met de limieten van je proxy/model.

  </Accordion>
  <Accordion title="Regels voor het vormgeven van proxyroutes">
    - Voor `api: "openai-completions"` op niet-systeemeigen eindpunten (elke niet-lege `baseUrl` waarvan de host niet `api.openai.com` is) dwingt OpenClaw `compat.supportsDeveloperRole: false` af om providerfouten 400 voor niet-ondersteunde `developer`-rollen te voorkomen.
    - OpenAI-compatibele proxyroutes slaan ook systeemeigen, uitsluitend voor OpenAI bedoelde verzoekvorming over: geen `service_tier`, geen Responses-`store`, geen Completions-`store`, geen hints voor de promptcache, geen vormgeving van payloads voor compatibiliteit met OpenAI-redenering en geen verborgen OpenClaw-toeschrijvingsheaders.
    - Voor OpenAI-compatibele Completions-proxy's die leveranciersspecifieke velden nodig hebben, stel je `agents.defaults.models["provider/model"].params.extra_body` (of `extraBody`) in om extra JSON samen te voegen in de hoofdtekst van het uitgaande verzoek.
    - Voor besturingselementen voor vLLM-chatsjablonen stel je `agents.defaults.models["provider/model"].params.chat_template_kwargs` in. De gebundelde vLLM-Plugin verzendt automatisch `enable_thinking: false` en `force_nonempty_content: true` voor `vllm/nemotron-3-*` wanneer het denkniveau van de sessie is uitgeschakeld.
    - Voor trage lokale modellen of externe LAN-/tailnethosts stel je `models.providers.<id>.timeoutSeconds` in. Dit verlengt de afhandeling van HTTP-verzoeken aan providermodellen, waaronder verbinding, headers, streaming van de hoofdtekst en de totale afbreking van beveiligd ophalen, zonder de time-out van de volledige agentruntime te verhogen. Als `agents.defaults.timeoutSeconds` of een uitvoeringsspecifieke time-out lager is, verhoog je dat plafond ook; providertime-outs kunnen de volledige uitvoering niet verlengen.
    - HTTP-aanroepen naar modelproviders staan fake-IP-DNS-antwoorden van Surge, Clash en sing-box in `198.18.0.0/15` en `fc00::/7` alleen toe voor de geconfigureerde providerhostnaam `baseUrl`. Aangepaste/lokale providereindpunten vertrouwen voor beveiligde modelverzoeken ook die exact geconfigureerde `scheme://host:port`-oorsprong, waaronder loopback-, LAN- en tailnethosts. Dit is geen nieuwe configuratieoptie; de `baseUrl` die je configureert, breidt het verzoekbeleid alleen voor die oorsprong uit. Toestemming voor fake-IP-hostnamen en vertrouwen in de exacte oorsprong zijn onafhankelijke mechanismen. Voor andere privé-, loopback-, link-local- en metadatabestemmingen en andere poorten blijft een expliciete inschakeling via `models.providers.<id>.request.allowPrivateNetwork: true` vereist. Stel `models.providers.<id>.request.allowPrivateNetwork: false` in om het vertrouwen in de exacte oorsprong uit te schakelen.
    - Als `baseUrl` leeg/weggelaten is, behoudt OpenClaw het standaardgedrag van OpenAI (dat wordt omgezet naar `api.openai.com`).
    - Voor de veiligheid wordt een expliciete `compat.supportsDeveloperRole: true` nog steeds overschreven op niet-systeemeigen `openai-completions`-eindpunten.
    - Voor `api: "anthropic-messages"` op niet-directe eindpunten (elke provider behalve de canonieke `anthropic`, of een aangepaste `models.providers.anthropic.baseUrl` waarvan de host geen openbaar `api.anthropic.com`-eindpunt is) onderdrukt OpenClaw impliciete Anthropic-bètaheaders zoals `claude-code-20250219`, `interleaved-thinking-2025-05-14` en OAuth-markeringen, zodat aangepaste Anthropic-compatibele proxy's niet-ondersteunde bètavlaggen niet afwijzen. Stel `models.providers.<id>.headers["anthropic-beta"]` expliciet in als je proxy specifieke bètafuncties nodig heeft.

  </Accordion>
</AccordionGroup>

## CLI-voorbeelden

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Zie ook: [Configuratie](/nl/gateway/configuration) voor volledige configuratievoorbeelden.

## Gerelateerd

- [Configuratiereferentie](/nl/gateway/config-agents#agent-defaults) - modelconfiguratiesleutels
- [Model-failover](/nl/concepts/model-failover) - fallbackketens en gedrag bij nieuwe pogingen
- [Modellen](/nl/concepts/models) - modelconfiguratie en aliassen
- [Providers](/nl/providers) - installatiehandleidingen per provider
