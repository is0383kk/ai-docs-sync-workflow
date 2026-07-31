---
read_when:
    - Je bouwt een lokale AI-CLI-backendplugin
    - Je wilt een backend registreren voor modelreferenties zoals acme-cli/model
    - Je moet een CLI van derden koppelen aan OpenClaws tekstuele fallback-runner
sidebarTitle: CLI backend plugins
summary: Bouw een Plugin die een lokale AI-CLI-backend registreert
title: CLI-backendplugins bouwen
x-i18n:
    generated_at: "2026-07-27T05:56:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

CLI-backendplugins laten OpenClaw een lokale AI-CLI aanroepen als backend voor
tekstinferentie. De backend verschijnt als providerprefix in modelverwijzingen:

```text
acme-cli/acme-large
```

Gebruik een CLI-backend wanneer de upstreamintegratie al beschikbaar is als lokale
opdracht, wanneer de CLI de lokale aanmeldingsstatus beheert, of als terugvaloptie wanneer API-
providers niet beschikbaar zijn.

<Info>
  Als de upstreamservice een normale HTTP-model-API aanbiedt, schrijf dan in plaats daarvan een
  [providerplugin](/nl/plugins/sdk-provider-plugins). Als de upstream-
  runtime volledige agentsessies, toolgebeurtenissen, compaction of de status van achtergrondtaken
  beheert, gebruik dan een [agentharnas](/nl/plugins/sdk-agent-harness).
</Info>

## Wat de plugin beheert

Een CLI-backendplugin heeft drie contracten:

| Contract             | Bestand                | Doel                                                      |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| Pakketentrypoint     | `package.json`         | Verwijst OpenClaw naar de runtimemodule van de plugin     |
| Manifesteigenaarschap | `openclaw.plugin.json` | Declareert de backend-id voordat de runtime wordt geladen |
| Runtimeregistratie   | `index.ts`             | Roept `api.registerCliBackend(...)` aan met standaardwaarden voor de opdracht |

Het manifest bevat detectiemetadata: het voert de CLI niet uit en registreert geen
runtimegedrag. Het runtimegedrag begint wanneer het pluginentrypoint
`api.registerCliBackend(...)` aanroept.

## Minimale backendplugin

<Steps>
  <Step title="Pakketmetadata maken">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    Gepubliceerde pakketten moeten gebouwde JavaScript-runtimebestanden bevatten. Als je bron-
    entrypoint `./src/index.ts` is, voeg dan `openclaw.runtimeExtensions` toe dat verwijst naar de
    gebouwde JavaScript-tegenhanger. Zie [Entrypoints](/nl/plugins/sdk-entrypoints).

  </Step>

  <Step title="Backendeigenaarschap declareren">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "Acme's lokale AI-CLI via OpenClaw uitvoeren",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` is de lijst met runtime-eigenaarschap; hiermee kan OpenClaw de
    plugin automatisch laden wanneer de modelselectie of `agentRuntime.id` `acme-cli` vermeldt.

    `setup.cliBackends` is het descriptorgerichte instellingsoppervlak. Voeg dit toe wanneer
    modeldetectie, onboarding of status de backend moet herkennen
    zonder de pluginruntime te laden. Gebruik `requiresRuntime: false` alleen wanneer
    die statische descriptors voldoende zijn voor de instelling.

  </Step>

  <Step title="De backend registreren">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Run Acme's local AI CLI through OpenClaw",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    De backend-id moet overeenkomen met de manifestvermelding `cliBackends`. De geregistreerde
    adapter is gezaghebbende plugincode; de OpenClaw-configuratie selecteert de backend,
    maar herschrijft het opdrachtcontract ervan niet.

  </Step>
</Steps>

## Configuratiestructuur

`CliBackendConfig` beschrijft hoe OpenClaw de CLI moet starten en parseren. Het
uitgewerkte voorbeeld hierboven gebruikt bewust dezelfde opdracht-, hervattings-, JSONL-,
modelalias-, sessie-, afbeeldings- en watchdogvelden als de gebundelde
`google-gemini-cli`-adapter:

| Veld                                                      | Gebruik                                                                           |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | Naam van het binaire bestand of absoluut opdrachtpad                              |
| `args`                                                    | Basis-argv voor nieuwe uitvoeringen                                               |
| `resumeArgs`                                              | Alternatieve argv voor hervatte sessies; ondersteunt `{sessionId}`                |
| `output` / `resumeOutput`                                 | Parser: `json`, `jsonl` of `text`                                                |
| `jsonlDialect`                                            | JSONL-gebeurtenisdialect: `claude-stream-json` of `gemini-stream-json`           |
| `liveSession`                                             | Modus voor langlevende CLI-processen (`claude-stdio`)                            |
| `input`                                                   | Prompttransport: `arg` of `stdin`                                                |
| `maxPromptArgChars`                                       | Maximale promptlengte voor de modus `arg` voordat wordt teruggevallen op stdin |
| `env` / `clearEnv`                                        | Extra omgevingsvariabelen om te injecteren of namen om vóór het starten te verwijderen |
| `modelArg`                                                | Vlag die vóór de model-id wordt gebruikt                                          |
| `modelAliases`                                            | OpenClaw-model-id's toewijzen aan CLI-eigen id's                                  |
| `sessionArgs`                                             | Hoe een sessie-id met `{sessionId}` moet worden doorgegeven                       |
| `sessionMode`                                             | `always`, `existing` of `none`                                                   |
| `sessionIdFields`                                         | JSON-velden die OpenClaw uit de CLI-uitvoer leest                                 |
| `systemPromptArg` / `systemPromptFileArg`                 | Transport van de systeemprompt                                                    |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | Transport voor configuratieoverschrijving van een systeempromptbestand (bijvoorbeeld `-c`) |
| `systemPromptMode`                                        | `append` of `replace`                                                             |
| `systemPromptWhen`                                        | `first`, `always` of `never`                                                     |
| `imageArg` / `imageMode`                                  | Vlag voor afbeeldingspaden en hoe meerdere afbeeldingen worden doorgegeven (`repeat` of `list`) |
| `imagePathScope`                                          | Waar klaargezette afbeeldingsbestanden vóór de overdracht staan: `temp` of `workspace` |
| `serialize`                                               | Uitvoeringen met dezelfde backend op volgorde houden                              |
| `reseedFromRawTranscriptWhenUncompacted`                  | Schakel begrensde herinitialisatie vanuit het onbewerkte transcript vóór compaction in voor veilige sessieresets |
| `reliability.watchdog`                                    | Afstelling van time-outs bij ontbrekende uitvoer, afzonderlijk voor nieuwe en hervatte uitvoeringen |

Geef de voorkeur aan de kleinste statische configuratie die bij de CLI past. Voeg alleen
plugincallbacks toe voor gedrag dat echt bij de backend hoort.

## Geavanceerde backendhooks

`CliBackendPlugin` kan ook het volgende definiëren:

| Hook                               | Gebruik                                                                     |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | De geregistreerde statische adapter normaliseren met runtimecontext         |
| `resolveExecutionArgs(ctx)`        | Aanvraaggebonden vlaggen toevoegen, zoals denkinspanning of isolatie van nevenvragen |
| `prepareExecution(ctx)`            | Tijdelijke bruggen voor authenticatie, configuratie of omgeving maken vóór het starten |
| `transformSystemPrompt(ctx)`       | Een laatste CLI-specifieke transformatie van de systeemprompt toepassen     |
| `textTransforms`                   | Tweerichtingsvervangingen voor prompts/uitvoer                              |
| `defaultAuthProfileId`             | Voorkeur geven aan een specifiek OpenClaw-authenticatieprofiel               |
| `authEpochMode`                    | Bepalen hoe authenticatiewijzigingen opgeslagen CLI-sessies ongeldig maken  |
| `nativeToolMode`                   | Declareren of systeemeigen tools afwezig, altijd ingeschakeld of door de host selecteerbaar zijn |
| `toolAvailabilityEnforcement`      | Declareren of exacte toolbeperkingen worden afgedwongen in argv of bij het klaarzetten van de uitvoering |
| `sideQuestionToolMode`             | Uitgeschakelde systeemeigen tools declareren voor `/btw`-nevenvragen       |
| `bundleMcp` / `bundleMcpMode`      | OpenClaws loopback-MCP-toolbrug inschakelen                                 |
| `ownsNativeCompaction`             | De backend beheert zijn eigen compaction — OpenClaw stelt deze uit           |
| `subscriptionAuthDispatch`         | Ingeschakelde ingebedde uitvoeringen met abonnementsreferenties worden via deze backend uitgevoerd |
| `runtimeArtifact`                  | Een scriptstarter beperken tot de volledige gebundelde pakketstructuur      |

Houd deze hooks in beheer van de provider. Voeg geen CLI-specifieke vertakkingen toe aan de kern wanneer
een backendhook het gedrag kan uitdrukken.

`prepareExecution(ctx)` ontvangt `ctx.contextTokenBudget`, de effectieve tokenlimiet
die voor de uitvoering is geselecteerd. Backends die native Compaction beheren, kunnen dat
budget koppelen aan hun CLI-specifieke startcontract.

`runtimeArtifact` is eigendom van de plugin. Het wordt
alleen geraadpleegd wanneer een live inferentiebeurt geverifieerde setupbevoegdheid aanmaakt of opnieuw valideert;
normale CLI-uitvoeringen vereisen dit niet. Een backend zonder deze declaratie kan
geen geverifieerde CLI-setupbevoegdheid aanmaken. Een declaratie van `bundled-package-tree` benoemt
de exacte eigenaar van `package.json` en vereist dat het pakket-entrypoint de
opdracht is. OpenClaw hasht de begrensde, volledige geïnstalleerde pakketstructuur, inclusief
geneste afhankelijkheden, en sluit bij fouten af voor omleidende symbolische koppelingen,
startprogramma's buiten het gedeclareerde pakket, declaraties van vereiste externe
afhankelijkheden, te grote structuren en onbekende scripts. Declareer dit alleen wanneer die
structuur de volledige inferentie-implementatie bevat; optionele toolintegraties
maken een externe implementatiegraaf niet veilig.

Als dezelfde backend ook een zelfstandig native uitvoerbaar bestand levert, vermeld dan de
canonieke basisnamen ervan in `nativeExecutableNames`. Andere native opdrachten blijven
niet-geverifieerd.

`ctx.executionMode` is `"agent"` voor normale beurten en `"side-question"` voor
tijdelijke `/btw`-aanroepen. Gebruik dit wanneer de CLI andere eenmalige vlaggen nodig heeft,
zoals het uitschakelen van native tools, sessiepersistentie of hervattingsgedrag voor
BTW. Als een backend normaal `nativeToolMode: "always-on"` heeft, maar de argv voor
nevenvragen die tools betrouwbaar uitschakelt, stel dan ook
`sideQuestionToolMode: "disabled"` in; anders sluit OpenClaw bij fouten af wanneer BTW
een CLI-uitvoering zonder tools vereist.

Stel `nativeToolMode: "selectable"` alleen in wanneer de backend elke
backend-native tool voor een afzonderlijke uitvoering kan uitschakelen. Beperkte uitvoeringen ontvangen een canoniek
contract: `ctx.toolAvailability.native` is de exacte backend-native lijst en
`ctx.toolAvailability.openClaw` is de exacte lijst met OpenClaw-toolnamen. De
host beperkt onafhankelijk de gegenereerde MCP-configuratie en toekenning tot die
OpenClaw-lijst; plugins mogen deze niet in de kern vertalen of transportvoorvoegsels toevoegen.

Declareer hoe de backend dat contract afdwingt:

- `toolAvailabilityEnforcement: "execution-args"` vereist
  `resolveExecutionArgs`. De hook moet conflicterende toolvlaggen vervangen, aanpassingsmogelijkheden
  uitschakelen die buiten de geselecteerde tools kunnen worden uitgevoerd, en
  afdwingende argv retourneren voor zowel nieuwe als hervatte uitvoeringen.
- `toolAvailabilityEnforcement: "prepare-execution"` vereist
  `prepareExecution`. De hook moet een exact beleid per uitvoering klaarzetten en
  `toolAvailabilityEnforced: true` retourneren; ontbrekende bevestiging leidt tot afsluiten bij fouten en
  OpenClaw ruimt de klaargezette resources vóór het starten op.

Runtimebeperkingen zoals Cron `toolsAllow` worden door
OpenClaw genormaliseerd en per groep uitgebreid voordat dit contract wordt opgebouwd. Native tools worden uitgeschakeld en een
backend zonder een volledig gedeclareerd handhavingspad faalt vóór uitvoering.

Plugins die zijn gebouwd tegen `v2026.7.2-beta.1` tot en met `v2026.7.2-beta.3` kunnen nog steeds
de verouderde projectie van transportnamen `ctx.toolAvailability.mcp` lezen en
mogen `toolAvailabilityEnforcement` weglaten wanneer een selecteerbare backend
`resolveExecutionArgs` implementeert. OpenClaw herkent dat uitgebrachte bètapad aan de
vereiste `openclaw.build.openclawVersion`-metadata van het pluginpakket en
behoudt het gedurende de `2026.8.x`-reeks. Nieuwe en bijgewerkte plugins moeten canonieke
`ctx.toolAvailability.openClaw`-namen gebruiken en
`toolAvailabilityEnforcement: "execution-args"` expliciet declareren; het
bètacompatibiliteitspad wordt na die periode verwijderd.

### `ownsNativeCompaction`: afzien van OpenClaw Compaction

Als je backend een agent uitvoert die zijn **eigen** transcript comprimeert, stel dan
`ownsNativeCompaction: true` in, zodat de beveiligende samenvatter van OpenClaw nooit
op de sessies ervan wordt uitgevoerd: de CLI-Compaction-levenscyclus doet niets en de
beurt gaat verder. `claude-cli` declareert dit omdat Claude Code intern comprimeert
zonder harness-endpoint. Native harness-sessies zoals Codex
blijven in plaats daarvan naar hun harness-Compaction-endpoint routeren.

**Declareer dit alleen wanneer aan alle volgende voorwaarden wordt voldaan**, anders kan een uitgestelde
sessie die het budget overschrijdt boven het budget blijven of verouderd raken (OpenClaw
herstelt deze niet langer):

- de backend comprimeert of begrenst zijn eigen transcript betrouwbaar wanneer het
  venster bijna vol is;
- de backend bewaart een hervatbare sessie, zodat de gecomprimeerde toestand tussen beurten behouden blijft
  (bijvoorbeeld `--resume` / `--session-id`);
- het is geen native harness-Compaction-sessie: overeenkomende `agentHarnessId`-sessies
  worden in plaats daarvan naar het harness-endpoint gerouteerd.

## MCP-toolbridge

CLI-backends ontvangen standaard geen OpenClaw-tools. Als de CLI
een MCP-configuratie kan verwerken, meld je dan expliciet aan:

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

Ondersteunde bridgemodi:

| Modus                     | Gebruik                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `claude-config-file`     | CLI's die een MCP-configuratiebestand accepteren                              |
| `codex-config-overrides` | CLI's die configuratieoverschrijvingen in argv accepteren                        |
| `gemini-system-settings` | CLI's die MCP-instellingen uit hun map met systeeminstellingen lezen |

Schakel de bridge alleen in wanneer de CLI deze daadwerkelijk kan verwerken. Als de CLI
een eigen ingebouwde toollaag heeft die niet kan worden uitgeschakeld, stel dan `nativeToolMode:
"always-on"` in, zodat OpenClaw bij fouten kan afsluiten wanneer een aanroeper geen native
tools vereist. Als elke native tool per uitvoering kan worden uitgeschakeld, gebruik dan `"selectable"` met het
bovenstaande `resolveExecutionArgs`-contract.

## De backend selecteren

Gebruikers selecteren een zelfstandige backend via het modelrefvoorvoegsel ervan. Een backend die
een canonieke `modelProvider` declareert, kan in plaats daarvan worden geselecteerd via de
`agentRuntime.id` van dat providermodel. De adaptermechanica blijft in de plugin:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

Plaats aanmeldgegevens in OpenClaw-authenticatieprofielen of configuratie die eigendom is van de plugin. Zorg ervoor dat de
geregistreerde opdracht zich in het `PATH` van de Gateway-service bevindt; implementaties die een
ander pad of andere argv nodig hebben, moeten de pluginregistratie wijzigen of omwikkelen.

## Verificatie

Voeg voor gebundelde plugins een gerichte test toe rond de builder en setupregistratie
en voer daarna de gerichte testbaan van de plugin uit:

```bash
pnpm test extensions/acme-cli
```

Verifieer voor lokale of geïnstalleerde plugins de detectie en één echte modeluitvoering:

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "antwoord exact: backend ok" --model acme-cli/acme-large
```

Als de backend afbeeldingen of MCP ondersteunt, voeg dan een live rooktest toe die deze
paden met de echte CLI aantoont. Vertrouw niet op statische inspectie voor prompt-, afbeeldings-,
MCP- of sessiehervattingsgedrag.

## Controlelijst

<Check>`package.json` heeft `openclaw.extensions` en gebouwde runtime-entrypoints voor gepubliceerde pakketten</Check>
<Check>`openclaw.plugin.json` declareert `cliBackends` en doelbewuste `activation.onStartup`</Check>
<Check>`setup.cliBackends` is aanwezig wanneer setup/modeldetectie de backend koud moet kunnen zien</Check>
<Check>`api.registerCliBackend(...)` gebruikt dezelfde backend-id als het manifest</Check>
<Check>Het backendmodelvoorvoegsel of de modelgebonden `agentRuntime.id` selecteert de registratie</Check>
<Check>Instellingen voor sessie, systeemprompt, afbeelding en uitvoerparser komen overeen met het echte CLI-contract</Check>
<Check>Gerichte tests en ten minste één live CLI-rooktest tonen het backendpad aan</Check>

## Gerelateerd

- [CLI-backends](/nl/gateway/cli-backends) - runtimeselectie en gedrag
- [Plugins bouwen](/nl/plugins/building-plugins) - basisprincipes van pakketten en manifesten
- [Overzicht van de Plugin-SDK](/nl/plugins/sdk-overview) - API-referentie voor registratie
- [Pluginmanifest](/nl/plugins/manifest) - `cliBackends` en setupbeschrijvingen
- [Agent-harness](/nl/plugins/sdk-agent-harness) - volledige externe agentruntimes
