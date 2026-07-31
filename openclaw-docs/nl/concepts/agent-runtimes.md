---
read_when:
    - Je kiest tussen OpenClaw, Codex, ACP of een andere native agentruntime
    - Je bent in de war door provider-/model-/runtimelabels in de status of configuratie
    - Je documenteert gelijkwaardige ondersteuning voor een native harness
summary: Hoe OpenClaw modelproviders, modellen, kanalen en agentruntimes van elkaar scheidt
title: Agentruntimes
x-i18n:
    generated_at: "2026-07-27T05:42:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 980d112946535df1566f2df4e3e71abacc2b073b51717c1e85fbb678691d39cb
    source_path: concepts/agent-runtimes.md
    workflow: 16
---

Een **agentruntime** beheert één voorbereide modellus: deze ontvangt de prompt,
stuurt de modeluitvoer aan, verwerkt native toolaanroepen en retourneert de voltooide beurt
aan OpenClaw.

Runtimes worden gemakkelijk verward met providers omdat beide in de buurt van de modelconfiguratie
voorkomen. Het zijn verschillende lagen:

| Laag          | Voorbeelden                                   | Betekenis                                                                   |
| ------------- | -------------------------------------------- | --------------------------------------------------------------------------- |
| Provider      | `anthropic`, `github-copilot`, `openai`      | Hoe OpenClaw authenticeert, modellen ontdekt en modelreferenties benoemt.   |
| Model         | `claude-opus-4-6`, `gpt-5.6-sol`             | Het model dat voor de agentbeurt is geselecteerd.                           |
| Agentruntime  | `claude-cli`, `codex`, `copilot`, `openclaw` | De onderliggende lus of backend die de voorbereide beurt uitvoert.          |
| Kanaal        | Discord, Slack, Telegram, WhatsApp           | Waar berichten OpenClaw binnenkomen en verlaten.                            |

Een **harnas** is de implementatie die een agentruntime levert (codeterm).
Het meegeleverde Codex-harnas implementeert bijvoorbeeld de runtime `codex`.
De openbare configuratie gebruikt `agentRuntime.id` voor provider- of modelvermeldingen; runtimesleutels
voor de volledige agent zijn verouderd en worden genegeerd. `openclaw doctor --fix` verwijdert oude
runtimevastleggingen voor de volledige agent en herschrijft verouderde runtimemodelreferenties naar canonieke
provider-/modelreferenties, plus waar nodig runtimebeleid op modelniveau.

Twee runtimefamilies:

- **Ingebedde harnassen** worden uitgevoerd binnen de voorbereide agentlus van OpenClaw: de
  ingebouwde runtime `openclaw`, plus geregistreerde Plugin-harnassen zoals
  `codex` en `copilot`.
- **CLI-backends** voeren een lokaal CLI-proces uit en houden daarbij de modelreferentie
  canoniek. `anthropic/claude-opus-5` met een modelgebonden
  `agentRuntime.id: "claude-cli"` betekent bijvoorbeeld: "selecteer het Anthropic-model en voer het uit
  via Claude CLI." `claude-cli` is geen id van een ingebed harnas en mag niet
  aan de selectie van AgentHarness worden doorgegeven.

Het harnas `copilot` is een afzonderlijk, optioneel extern Plugin-harnas voor de
GitHub Copilot CLI; zie [GitHub Copilot-agentruntime](/nl/plugins/copilot) voor
de gebruikersgerichte keuze tussen PI, Codex en de GitHub Copilot-agentruntime.

## Codex-oppervlakken

Meerdere oppervlakken delen de naam Codex:

| Oppervlak                                        | OpenClaw-naam/configuratie            | Functie                                                                                                                 |
| ------------------------------------------------ | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| Native runtime van Codex app-server              | `openai/*`-modelreferenties                | Voert ingebedde OpenAI-agentbeurten uit via Codex app-server. Dit is de gebruikelijke configuratie voor een ChatGPT-/Codex-abonnement. |
| Codex OAuth-authenticatieprofielen               | `openai` OAuth-profielen              | Slaat de authenticatie van het ChatGPT-/Codex-abonnement op die door het Codex app-server-harnas wordt gebruikt.         |
| Codex ACP-adapter                                | `runtime: "acp"`, `agentId: "codex"` | Voert Codex uit via het externe ACP-/acpx-besturingsvlak. Gebruik dit alleen wanneer expliciet om ACP/acpx wordt gevraagd. |
| Native Codex-chatbesturingsopdrachten            | `/codex ...`                         | Koppelt, hervat, stuurt, stopt en inspecteert Codex app-server-threads vanuit de chat.                                   |
| OpenAI Platform API-route voor niet-agentoppervlakken | `openai/*` plus authenticatie met API-sleutel | Directe OpenAI-API's, zoals afbeeldingen, embeddings, spraak en realtime.                                                |

Deze oppervlakken zijn bewust onafhankelijk. Door de Plugin `codex` in te schakelen,
worden native app-serverfuncties beschikbaar; `openclaw doctor --fix` beheert
het herstel van verouderde Codex-routes en het opschonen van achtergebleven sessievastleggingen. Het selecteren van `openai/*`
voor een agentmodel betekent nu "voer dit uit via Codex", tenzij een niet-agentgebonden
OpenAI API-oppervlak wordt gebruikt.

De gebruikelijke configuratie voor een ChatGPT-/Codex-abonnement gebruikt Codex OAuth voor authenticatie,
maar behoudt `openai/*` als modelreferentie en selecteert de runtime `codex`:

```json5
{
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Dit betekent dat OpenClaw een OpenAI-modelreferentie selecteert en vervolgens de runtime van Codex
app-server vraagt de ingebedde agentbeurt uit te voeren. Het betekent niet "gebruik API-
facturering" en ook niet dat het kanaal, de catalogus van modelproviders of
de OpenClaw-sessieopslag Codex wordt.

Wanneer de meegeleverde Plugin `codex` is ingeschakeld, gebruik je het native opdrachtoppervlak `/codex`
(`/codex bind`, `/codex threads`, `/codex resume`, `/codex steer`,
`/codex stop`) voor Codex-besturing met natuurlijke taal in plaats van ACP. Gebruik ACP voor
Codex alleen wanneer de gebruiker expliciet om ACP/acpx vraagt of het pad van de ACP-
adapter test. Claude Code, Gemini CLI, OpenCode, Cursor en vergelijkbare externe
harnassen blijven ACP gebruiken.

Beslisboom:

1. **Codex koppelen/besturen/thread/hervatten/sturen/stoppen** -> native opdrachtoppervlak `/codex` wanneer de meegeleverde Plugin `codex` is ingeschakeld.
2. **Codex als ingebedde runtime** of de normale, door een abonnement ondersteunde Codex-agentervaring -> `openai/<model>`.
3. **OpenClaw expliciet gekozen voor een OpenAI-model** -> behoud `openai/<model>` als modelreferentie en stel het runtimebeleid voor provider/model in op `agentRuntime.id: "openclaw"`. Een geselecteerd OAuth-profiel `openai` wordt intern gerouteerd via het Codex-authenticatietransport van OpenClaw.
4. **Verouderde Codex-modelreferenties in de configuratie** -> herstel met `openclaw doctor --fix` naar `openai/<model>`; doctor behoudt de Codex-authenticatieroute door waar de oude modelreferentie dit impliceerde `agentRuntime.id: "codex"` op provider-/modelniveau toe te voegen. Verouderde **`codex-cli/*`**-modelreferenties worden hersteld naar dezelfde `openai/<model>`-route van Codex app-server; OpenClaw bevat niet langer een meegeleverde Codex CLI-backend.
5. **Expliciet gevraagd om ACP, acpx of de Codex ACP-adapter** -> `runtime: "acp"` en `agentId: "codex"`.
6. **Claude Code, Gemini CLI, OpenCode, Cursor, Droid of een ander extern harnas** -> ACP/acpx, niet de native subagentruntime.

| Je bedoelt...                           | Gebruik...                                    |
| --------------------------------------- | -------------------------------------------- |
| Chat-/threadbesturing van Codex app-server | `/codex ...` uit de meegeleverde Plugin `codex` |
| Ingebedde agentruntime van Codex app-server | `openai/*`-agentmodelreferenties                  |
| OpenAI Codex OAuth                      | `openai` OAuth-profielen                      |
| Claude Code of een ander extern harnas  | ACP/acpx                                     |

Zie [OpenAI](/nl/providers/openai) en
[Modelproviders](/nl/concepts/model-providers) voor de opsplitsing van het voorvoegsel van de OpenAI-familie. Zie voor het
ondersteuningscontract van de Codex-runtime [Codex-harnasruntime](/nl/plugins/codex-harness-runtime#v1-support-contract).

## Runtime-eigenaarschap

Verschillende runtimes beheren verschillende delen van de lus:

| Oppervlak                   | Ingebed in OpenClaw                            | Codex app-server                                                            |
| --------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------- |
| Eigenaar van de modellus    | OpenClaw, via de ingebedde OpenClaw-runner     | Codex app-server                                                            |
| Canonieke threadstatus      | OpenClaw-transcript                            | Codex-thread, plus een spiegel van het OpenClaw-transcript                  |
| Dynamische OpenClaw-tools   | Native OpenClaw-toollus                        | Overbrugd via de Codex-adapter                                              |
| Native shell- en bestandstools | OpenClaw-pad                               | Codex-native tools, waar ondersteund overbrugd via native hooks             |
| Contextengine               | Native contextassemblage van OpenClaw          | OpenClaw projecteert de samengestelde context in de Codex-beurt             |
| Compaction                  | OpenClaw of de geselecteerde contextengine     | Codex-native Compaction, met OpenClaw-meldingen en spiegelonderhoud         |
| Kanaalaflevering            | OpenClaw                                       | OpenClaw                                                                    |

Ontwerpregel: als OpenClaw eigenaar is van het oppervlak, kan het normaal gedrag van Plugin-hooks
bieden. Als de native runtime eigenaar is van het oppervlak, heeft OpenClaw runtime-
gebeurtenissen of native hooks nodig. Als de native runtime eigenaar is van de canonieke threadstatus,
spiegelt OpenClaw de context en projecteert deze, in plaats van niet-ondersteunde
interne onderdelen te herschrijven.

## Runtimeselectie

OpenClaw bepaalt na het oplossen van de provider en het model een ingebedde runtime, in
deze volgorde:

1. **Modelgebonden runtimebeleid** heeft voorrang. Dit bevindt zich in een geconfigureerde provider-
   modelvermelding of in `agents.defaults.models["provider/model"].agentRuntime`
   / `agents.entries.*.models["provider/model"].agentRuntime`. Een provider-
   wildcard zoals `agents.defaults.models["vllm/*"].agentRuntime` wordt
   na exact modelbeleid toegepast, zodat dynamisch ontdekte providermodellen
   één runtime kunnen delen zonder exacte uitzonderingen per model te overschrijven.
2. **Providergebonden runtimebeleid**: `models.providers.<provider>.agentRuntime`.
3. **Modus `auto`**: geregistreerde Plugin-runtimes kunnen ondersteunde provider-/modelparen claimen.
4. Als niets de beurt claimt in de modus `auto`, valt OpenClaw terug op
   `openclaw` als compatibiliteitsruntime. Gebruik een expliciete runtime-id wanneer
   de uitvoering strikt moet zijn.

Runtimevastleggingen voor de volledige sessie en de volledige agent worden genegeerd: `OPENCLAW_AGENT_RUNTIME`,
sessiestatus `agentHarnessId`/`agentRuntimeOverride`, `agents.defaults.agentRuntime`
en `agents.entries.*.agentRuntime`. Voer `openclaw doctor --fix` uit om achtergebleven
runtimeconfiguratie voor de volledige agent te verwijderen en verouderde runtimemodelreferenties te converteren waar de bedoeling
behouden kan blijven.

Expliciete Plugin-runtimes voor provider/model weigeren standaard: `agentRuntime.id: "codex"`
voor een provider of model betekent Codex, of een duidelijke selectie-/runtimefout; dit wordt
nooit stilzwijgend teruggerouteerd naar OpenClaw. Alleen `auto` mag een niet-overeenkomende
beurt naar OpenClaw routeren.

Aliassen voor CLI-backends verschillen van id's van ingebedde harnassen. Voorkeursvorm voor Claude CLI:

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

Verouderde referenties zoals `claude-cli/claude-opus-4-7` blijven ondersteund voor
compatibiliteit, maar nieuwe configuratie moet provider/model canoniek houden en
de uitvoeringsbackend in het runtimebeleid voor provider/model plaatsen.

Verouderde `codex-cli/*`-referenties zijn anders: doctor migreert ze naar `openai/*`, zodat
ze via het Codex app-server-harnas worden uitgevoerd in plaats van een Codex
CLI-backend te behouden.

De modus `auto` is bewust conservatief voor de meeste providers. OpenAI-agentmodellen
vormen de uitzondering: zowel een niet-ingestelde runtime als `auto` wordt omgezet naar het Codex-
harnas. Expliciete OpenClaw-runtimeconfiguratie blijft een optionele compatibiliteitsroute
voor `openai/*`-agentbeurten; wanneer deze is gekoppeld aan een geselecteerd OAuth-
profiel `openai`, routeert OpenClaw dat pad intern via het Codex-authenticatietransport,
terwijl `openai/*` de openbare modelreferentie blijft. Achtergebleven OpenAI-
runtimesessievastleggingen worden door de runtimeselectie genegeerd en kunnen worden opgeschoond met
`openclaw doctor --fix`.

Als `openclaw doctor` waarschuwt dat de Plugin `codex` is ingeschakeld terwijl er nog verouderde
Codex-modelverwijzingen in de configuratie staan, behandel dit dan als de status van een verouderde route en voer
`openclaw doctor --fix` uit om deze met de Codex-runtime te herschrijven naar `openai/*`.

## GitHub Copilot-agentruntime

De externe Plugin `@openclaw/copilot` registreert een optionele `copilot`-runtime
die wordt ondersteund door de GitHub Copilot CLI (`@github/copilot-sdk`). Deze claimt de
canonieke abonnementsprovider `github-copilot` en wordt **nooit** geselecteerd door
`auto`. Schakel deze per model of per provider in via `agentRuntime.id`:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/gpt-5.5",
      models: {
        "github-copilot/gpt-5.5": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

De harness claimt zijn provider, runtime, CLI-sessiesleutel en voorvoegsel voor het
authenticatieprofiel in `extensions/copilot/doctor-contract-api.ts`, dat door `openclaw doctor`
automatisch wordt geladen. Zie [GitHub Copilot-agentruntime](/nl/plugins/copilot) voor configuratie, authenticatie, spiegeling van transcripten, Compaction, het
declaratieve doctor-contract en de bredere SDK-keuze tussen PI, Codex en Copilot.

## Compatibiliteitscontract

Wanneer een runtime niet OpenClaw is, moet de documentatie ervan vermelden welke OpenClaw-oppervlakken
worden ondersteund:

| Vraag                                  | Waarom dit belangrijk is                                                                            |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Wie beheert de modellus?               | Bepaalt waar nieuwe pogingen, voortzetting van tools en beslissingen over het definitieve antwoord plaatsvinden. |
| Wie beheert de canonieke threadgeschiedenis? | Bepaalt of OpenClaw de geschiedenis kan bewerken of alleen kan spiegelen.                         |
| Werken dynamische OpenClaw-tools?      | Berichten, sessies, Cron en tools die OpenClaw beheert, zijn hiervan afhankelijk.                    |
| Werken dynamische toolhooks?           | Plugins verwachten `before_tool_call`, `after_tool_call` en middleware rond tools die OpenClaw beheert. |
| Werken native toolhooks?               | Shell-, patch- en door de runtime beheerde tools hebben native hookondersteuning nodig voor beleid en observatie. |
| Wordt de levenscyclus van de contextengine uitgevoerd? | Geheugen- en contextplugins zijn afhankelijk van de levenscyclus voor samenstellen, opnemen, na de beurt en Compaction. |
| Welke Compaction-gegevens worden beschikbaar gesteld? | Sommige Plugins hebben alleen meldingen nodig; andere hebben metagegevens over behouden/verwijderde inhoud nodig. |
| Wat wordt bewust niet ondersteund?     | Gebruikers moeten niet uitgaan van gelijkwaardigheid met OpenClaw wanneer de native runtime meer status beheert. |

Het ondersteuningscontract voor de Codex-runtime wordt beschreven in
[Codex-harnessruntime](/nl/plugins/codex-harness-runtime#v1-support-contract).

## Statuslabels

Statusuitvoer kan zowel de labels `Execution` als `Runtime` tonen. Lees deze als
diagnostiek, niet als providernamen:

- Een modelverwijzing zoals `openai/gpt-5.6-sol` is de geselecteerde provider/het geselecteerde model.
- Een runtime-id zoals `codex` is de lus die de beurt uitvoert.
- Een kanaallabel zoals Telegram of Discord geeft aan waar het gesprek plaatsvindt.

Als een uitvoering een onverwachte runtime toont, controleer dan eerst het runtimebeleid van de
geselecteerde provider/het geselecteerde model. Verouderde runtimepinnen voor sessies bepalen de routering niet meer.

## Gerelateerd

- [Codex-harness](/nl/plugins/codex-harness)
- [Codex-harnessruntime](/nl/plugins/codex-harness-runtime)
- [GitHub Copilot-agentruntime](/nl/plugins/copilot)
- [OpenAI](/nl/providers/openai)
- [Plugins voor agentharnassen](/nl/plugins/sdk-agent-harness)
- [Agentlus](/nl/concepts/agent-loop)
- [Modellen](/nl/concepts/models)
- [Status](/nl/cli/status)
