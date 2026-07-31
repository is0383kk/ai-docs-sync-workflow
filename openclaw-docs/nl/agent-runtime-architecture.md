---
summary: 'Hoe OpenClaw de ingebouwde agentruntime structureert: code-indeling, grenzen, resourcemanifesten en runtimeselectie.'
title: Architectuur van de agentruntime
x-i18n:
    generated_at: "2026-07-27T04:55:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e09ff21b4369a7c102db51e4458ad3ba1e86c9fe43a3a8bff72eef1713d2d51
    source_path: agent-runtime-architecture.md
    workflow: 16
---

OpenClaw beheert de ingebouwde agentruntime. Runtimecode bevindt zich onder `src/agents/`, model-/providertransport bevindt zich onder `src/llm/` en contracten voor plugins worden beschikbaar gesteld via `openclaw/plugin-sdk/*`-barrels.

## Runtime-indeling

| Pad                                 | Beheert                                                                                                                                                                                                                   |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/agents/embedded-agent-runner/` | Ingebouwde poginglus (`run.ts`, `run/`), modelselectie en providernormalisatie (`model*.ts`), aanvraagparameters per provider (`extra-params.*`), Compaction en koppeling van transcript en sessie.                            |
| `src/agents/sessions/`              | Sessiepersistentie (`session-manager.ts`), resourcedetectie (`package-manager.ts`, `resource-loader.ts`), laden van `extensions` binnen de sessie, promptsjablonen, Skills, thema's en door de TUI ondersteunde toolrenderers (`tools/`). |
| `packages/agent-core/`              | Herbruikbare agentkern (`@openclaw/agent-core`): agentlus, harnastypen, berichten, Compaction-helpers, promptsjablonen, Skills en contracten voor sessieopslag.                                                           |
| `src/agents/runtime/`               | OpenClaw-facade die `@openclaw/agent-core` verbindt met de LLM-runtime van de plugin-SDK en deze opnieuw exporteert, samen met lokale proxyhulpprogramma's.                                                                                             |
| `src/agents/agent-tools*.ts`        | Tooldefinities, parameterschema's, toolbeleid, adapters vóór/na toolaanroepen en bewerkingstools voor host/sandbox die door OpenClaw worden beheerd.                                                                                            |
| `src/agents/agent-hooks/`           | Ingebouwde runtimehooks: Compaction-beveiliging, Compaction-instructies, contextopschoning.                                                                                                                                   |
| `src/agents/harness/`               | Harnasregister, selectiebeleid en levenscyclus voor de ingebouwde en door plugins geregistreerde harnassen.                                                                                                                       |
| `src/llm/`                          | Model-/providerregister, transporthelpers en providerspecifieke streamimplementaties (`src/llm/providers/`).                                                                                                          |

## Grenzen

De kern roept de ingebouwde runtime aan via OpenClaw-modules en SDK-barrels; er zijn geen externe pakketten voor agentframeworks meer. Plugins gebruiken gedocumenteerde `openclaw/plugin-sdk/*`-toegangspunten en importeren geen interne onderdelen van `src/**`.

`@earendil-works/pi-tui` blijft een afhankelijkheid van derden: een toolkit voor terminalcomponenten die wordt gebruikt door de lokale TUI en toolrenderers voor sessies. Internalisering ervan zou een afzonderlijke vendoringinspanning zijn.

## Manifesten

Resourcepakketten declareren OpenClaw-resources in `package.json`-metadata. Vermeldingen zijn bestandspaden of globs relatief ten opzichte van de pakketroot:

```json
{
  "openclaw": {
    "extensions": ["extensions/index.ts"],
    "skills": ["skills/*.md"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

Resourcetypen die niet in een manifest staan, vallen terug op detectie van conventionele mappen `extensions/`, `skills/`, `prompts/` en `themes/`.

## Runtimeselectie

- De id van de ingebouwde runtime is `openclaw`. De verouderde alias `pi` wordt genormaliseerd naar `openclaw`; `codex-app-server` wordt genormaliseerd naar `codex`.
- Pluginharnassen registreren aanvullende runtime-id's (bijvoorbeeld `codex`).
- Runtimebeleid is model-/providergebonden `agentRuntime.id`-configuratie (de modelvermelding heeft voorrang op de providervermelding). Niet ingesteld of `default` wordt omgezet naar `auto`.
- `auto` selecteert een geregistreerd pluginharnas dat de effectieve providerroute ondersteunt, en anders de ingebouwde OpenClaw-runtime. Alleen een provider- of modelvoorvoegsel selecteert nooit een harnas.
- OpenAI mag `codex` alleen impliciet selecteren voor een exacte officiële HTTPS-route voor Platform Responses of ChatGPT Responses zonder een zelf opgegeven aanvraagoverschrijving. Completions-adapters, aangepaste eindpunten en routes met zelf opgegeven aanvraaggedrag blijven op `openclaw`; officiële HTTP-eindpunten met platte tekst worden geweigerd. Zie [Impliciete OpenAI-agentruntime](/nl/providers/openai#implicit-agent-runtime).

## Generaties van modelruntimes

Bij het starten van de Gateway en bij publicatie van configuratie, plugins of authenticatie wordt per geconfigureerde agent één voorbereide modelruntimegeneratie gebouwd. Elke generatie beheert de gedetecteerde authenticatiesjabloon, het modelregister en de geprojecteerde modelcatalogus als één atomische momentopname. Agentuitvoeringen maken vanuit die momentopname afsplitsingen van veranderlijke authenticatie- en registeropslag; browse-, status-, Cron-, doctor-, TUI-, PDF- en afbeeldingspaden lezen de gepubliceerde catalogus in plaats van de bestandssysteemdetectie te herhalen.

Zelfstandige ingebedde runtimes publiceren dezelfde momentopnamestructuur op hun activeringsgrens. Een mislukte of verouderde generatie wordt nooit naast een nieuwere gedeeltelijke generatie aangeboden; de eigenaar van de levenscyclus moet eerst een volledige vervanging publiceren.

## Gerelateerd

- [Workflow van de OpenClaw-agentruntime](/nl/openclaw-agent-runtime)
- [Agentruntimes](/nl/concepts/agent-runtimes)
