---
read_when:
    - Je ziet een configuratiesleutel `.experimental` en wilt weten of deze stabiel is
    - Je wilt runtimefuncties in preview uitproberen zonder ze te verwarren met de normale standaardinstellingen
    - Je wilt één plek waar je de momenteel gedocumenteerde experimentele vlaggen kunt vinden
summary: Wat experimentele vlaggen in OpenClaw betekenen en welke momenteel zijn gedocumenteerd
title: Experimentele functies
x-i18n:
    generated_at: "2026-07-27T05:02:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6c14b74bbafce77c0d1e1358ad94053675c4aad9e26be78719f58e78f455c3a2
    source_path: concepts/experimental-features.md
    workflow: 16
---

Experimentele functies zijn previewonderdelen achter expliciete vlaggen. Ze moeten in de praktijk meer worden beproefd voordat ze een stabiele standaardinstelling of een langdurig contract krijgen.

- Standaard uitgeschakeld, tenzij documentatie een beperkte regel voor automatische configuratie beschrijft.
- Vorm en gedrag kunnen sneller veranderen dan stabiele configuratie.
- Geef de voorkeur aan een stabiele werkwijze als die al bestaat.
- Rol pas breed uit nadat je eerst in een kleinere omgeving hebt getest.

## Momenteel gedocumenteerde vlaggen

| Onderdeel                | Sleutel                                                                                        | Gebruik dit wanneer                                                                                                                    | Meer                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Lokale modelruntime      | `agents.defaults.experimental.localModelLean`, `agents.entries.*.experimental.localModelLean` | Een kleinere of strengere lokale backend vastloopt op het volledige standaardtoolaanbod van OpenClaw                                  | [Lokale modellen](/nl/gateway/local-models)                                               |
| Codex-harnas             | `plugins.entries.codex.config.appServer.experimental.sandboxExecServer`                       | Je native Codex app-server 0.132.0 of nieuwer wilt richten op een door een OpenClaw-sandbox ondersteunde exec-server in plaats van Code Mode uit te schakelen | [Naslaginformatie over het Codex-harnas](/nl/plugins/codex-harness-reference#sandboxed-native-execution) |
| Tool voor gestructureerde planning | `tools.experimental.planTool`                                                                 | Je de gestructureerde tool `update_plan` beschikbaar wilt maken voor het bijhouden van werk met meerdere stappen in compatibele runtimes en UI's | [Naslaginformatie over Gateway-configuratie](/nl/gateway/config-tools#toolsexperimental)  |
| Code Mode                | `tools.codeMode.enabled`                                                                      | Je compacte, door code georkestreerde toegang wilt tot een verborgen OpenClaw-toolcatalogus                                            | [Code Mode](/nl/tools/code-mode)                                                          |
| Swarm                    | `tools.swarm.enabled`                                                                         | Je wilt dat Code Mode-scripts begrensde groepen subagenten parallel orkestreren                                                        | [Swarm](/tools/swarm)                                                                  |

## Labs in de Control UI

Open **Settings → Agents & Tools → Labs** om experimenten te beheren die een
schakelaar in de Control UI hebben. Als je een lab inschakelt of uitschakelt,
wordt de canonieke Gateway-configuratie onmiddellijk aangepast; de pagina toont
alleen een hint om opnieuw te starten wanneer een functie dit vereist.

Code Mode en Swarm zijn de momenteel uitgebrachte vermeldingen in Labs. Beide
schakelaars schrijven bestaande gevalideerde configuratiesleutels en worden
normaal gesproken zonder herstart van de Gateway van kracht voor toekomstige
agentruns.

## Lean-modus voor lokale modellen

`agents.defaults.experimental.localModelLean: true` verwijdert bij elke beurt zware optionele tools uit het directe aanbod van de agent: `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` en `pdf`. Expliciet toegestane tools of tools die voor aflevering vereist zijn, blijven beschikbaar, hoewel Tool Search ze mogelijk catalogiseert in plaats van rechtstreeks beschikbaar te maken. Lean-modus stelt daarnaast plugin-/MCP-/clientcatalogi standaard in op gestructureerde Tool Search (`tool_search`, `tool_describe`, `tool_call`) wanneer `tools.toolSearch` nog niet is ingesteld. Gebruik `agents.entries.*.experimental.localModelLean` om dit tot één agent te beperken.

Tijdens de onboarding stelt een geverifieerde inferentieroute via `ollama` of `lmstudio` automatisch `agents.defaults.experimental.localModelLean: true` in wanneer die waarde ontbreekt. OpenClaw registreert dat de instelling afkomstig is van de onboarding, zodat een later geverifieerde niet-lokale route alleen de automatische instelling opheft. Een expliciet geconfigureerde `true` of `false` blijft behouden. Andere zelfgehoste en OpenAI-compatibele providers worden niet afgeleid uit modelnamen of URL's.

Als je Tool Search al globaal afstemt, laat OpenClaw die configuratie ongemoeid. Stel `tools.toolSearch: false` in om de standaardinstelling voor Tool Search van de lean-modus niet te gebruiken.

In de gestructureerde modus `tools` houden lean-runs `exec` rechtstreeks zichtbaar naast de bedieningselementen van Tool Search, zodat voor programmeren afgestemde lokale modellen nog steeds hun vertrouwde shellroute kunnen kiezen. Dit verandert alleen de zichtbaarheid van het schema: het normale toolbeleid, sandboxing en goedkeuringen voor uitvoering blijven van toepassing. De expliciete modi `code` en `directory` behouden hun normale Compaction-gedrag.

### Waarom deze tools

Deze tools hebben de langste beschrijvingen, de breedste parametervormen of de grootste kans om een klein model af te leiden van het normale programmeer- en gesprekspad. Op een backend met een kleine context of een strengere OpenAI-compatibele backend is dat het verschil tussen:

- Toolschema's die in de prompt passen tegenover toolschema's die de gespreksgeschiedenis verdringen.
- Het model dat de juiste tool kiest tegenover het genereren van ongeldige toolaanroepen door te veel vergelijkbare schema's.
- De Chat Completions-adapter die binnen de limieten voor gestructureerde uitvoer blijft tegenover een 400-fout vanwege de grootte van de payload van de toolaanroep.

Door ze te verwijderen wordt alleen de directe toollijst korter. Het model beschikt nog steeds over `read`, `write`, `edit`, `exec`, `apply_patch`, beeldherkenning, zoeken/ophalen via het web (wanneer geconfigureerd), geheugen en sessie-/agenttools. Extra catalogi blijven bereikbaar via Tool Search, tenzij je `tools.toolSearch: false` instelt; met expliciete tooltoestemmingen kan een lean-agent weer toegang krijgen tot een ingekorte workflow.

### Wanneer je dit inschakelt

Schakel de lean-modus in zodra je hebt aangetoond dat het model met de Gateway kan communiceren, maar volledige agentbeurten zich onjuist gedragen:

1. `openclaw infer model run --gateway --model <ref> --prompt "Reply with exactly: pong"` slaagt.
2. Een normale agentbeurt mislukt door ongeldige toolaanroepen, te grote prompts of doordat het model zijn tools negeert.
3. Het omschakelen van `localModelLean: true` verhelpt de fout.

### Wanneer je dit uitgeschakeld laat

Als je backend de volledige standaardruntime probleemloos verwerkt, laat je dit uitgeschakeld. Het is een tijdelijke oplossing voor lokale stacks die een kleiner toolaanbod nodig hebben, geen standaardinstelling voor gehoste modellen of lokale systemen met voldoende middelen.

Lean-modus vervangt `tools.profile`, `tools.allow`/`tools.deny` of de uitwijkmogelijkheid `compat.supportsTools: false` van het model niet. Voor een permanent beperkter toolaanbod voor een specifieke agent geef je de voorkeur aan die stabiele instellingen.

### Inschakelen

```json5
{
  agents: {
    defaults: {
      experimental: {
        localModelLean: true,
      },
    },
  },
}
```

Alleen voor één agent:

```json5
{
  agents: {
    list: [
      {
        id: "local",
        model: "lmstudio/gemma-4-e4b-it",
        experimental: {
          localModelLean: true,
        },
      },
    ],
  },
}
```

Start de Gateway opnieuw nadat je de vlag hebt gewijzigd. Lean-filtering verwijdert `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` en `pdf`, tenzij je ze expliciet behoudt met `tools.allow` of `tools.alsoAllow`; Tool Search kan behouden tools nog steeds catalogiseren in plaats van ze rechtstreeks beschikbaar te maken.

## Experimenteel betekent niet verborgen

Documentatie en het configuratiepad zelf moeten duidelijk aangeven dat een functie experimenteel is; deze status mag niet achter een ogenschijnlijk stabiele standaardinstelling worden verborgen.

## Gerelateerd

- [Functies](/nl/concepts/features)
- [Releasekanalen](/nl/install/development-channels)
