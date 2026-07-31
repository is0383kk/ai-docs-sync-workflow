---
read_when:
    - Je wilt een betrouwbare terugvaloptie wanneer API-providers uitvallen
    - Je voert lokale AI-CLI's uit en wilt ze hergebruiken
    - Je wilt de MCP-loopbackbridge voor toegang tot CLI-backendtools begrijpen
summary: 'CLI-backends: lokale AI-CLI als fallback met optionele MCP-toolbridge'
title: CLI-backends
x-i18n:
    generated_at: "2026-07-27T05:09:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce0427c587bf2a1e0a2ff24b5e76952eecae059e6f900af777b897b2d8d4210
    source_path: gateway/cli-backends.md
    workflow: 16
---

OpenClaw kan een lokale AI-CLI uitvoeren als een uitsluitend tekstuele fallback wanneer API-providers niet beschikbaar zijn, rate limits toepassen of zich onjuist gedragen. Dit is bewust conservatief:

- OpenClaw-tools worden niet rechtstreeks geïnjecteerd, maar een backend met `bundleMcp: true` kan Gateway-tools ontvangen via een loopback-MCP-bridge.
- JSONL-streaming voor CLI's die dit ondersteunen.
- Sessies worden ondersteund, zodat vervolgbeurten coherent blijven.
- Afbeeldingen worden doorgegeven als de CLI afbeeldingspaden accepteert.

Gebruik dit als vangnet voor tekstreacties die 'altijd werken', niet als primair pad. Gebruik in plaats daarvan [ACP-agents](/nl/tools/acp-agents) voor een volledige harness-runtime met ACP-sessiebediening, achtergrondtaken, koppeling aan threads/gesprekken en persistente externe codeersessies; CLI-backends zijn geen ACP.

<Tip>
  Een nieuwe backendplugin bouwen? Zie [CLI-backendplugins](/nl/plugins/cli-backend-plugins). Deze pagina behandelt het configureren en gebruiken van een reeds geregistreerde backend.
</Tip>

## Snel aan de slag

De gebundelde Anthropic-plugin registreert een standaardbackend `claude-cli`, zodat deze zonder configuratie werkt, afgezien van de vereiste dat Claude Code geïnstalleerd is en je bent ingelogd:

```bash
openclaw agent --agent main --message "hi" --model claude-cli/claude-sonnet-4-6
```

`main` is de standaard-agent-id wanneer geen expliciete lijst met agents is geconfigureerd; gebruik anders je eigen agent-id.

De Gateway-service moet de CLI in zijn `PATH` hebben. Als een implementatie een
niet-standaardpad naar het uitvoerbare bestand of niet-standaardargumenten nodig heeft, registreer die adapter dan in een
[CLI-backendplugin](/nl/plugins/cli-backend-plugins) in plaats van de startmechanismen
in `openclaw.json` te plaatsen.

OpenClaw laadt automatisch een gebundelde eigenaarplugin wanneer de modelselectie of een
modelgebonden `agentRuntime.id` naar de backend ervan verwijst.

## Gebruiken als fallback

Voeg de CLI-backend toe aan je lijst met fallbacks, zodat deze alleen wordt uitgevoerd wanneer primaire modellen mislukken:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
      },
    },
  },
}
```

Geconfigureerde fallbacks blijven beschikbaar wanneer de primaire provider mislukt (authenticatie, rate limits, time-outs), zelfs wanneer ze niet in `agents.defaults.modelPolicy.allow` staan. Voeg een CLI-backendmodel alleen aan dat beleid toe wanneer gebruikers het ook rechtstreeks moeten kunnen selecteren via `/model`, een sessie-override of `--model`. `agents.defaults.models` beheert alleen aliassen, parameters en metadata per model.

## Configuratie

Gebruikers kiezen een geregistreerde backend via het model- en runtimebeleid. Houd
de modelverwijzing canoniek en selecteer de CLI-runtime per model:

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

Referenties blijven in OpenClaw-authenticatieprofielen of de configuratie van de eigenaarplugin.
Mechanismen voor opdrachten, argv, omgeving, parsing, sessies, afbeeldingen en watchdogs zijn
plugincode die met `api.registerCliBackend(...)` is geregistreerd.

## Hoe het werkt

1. Selecteert een backend op basis van het providerprefix (`claude-cli/...`).
2. Bouwt een systeemprompt met dezelfde OpenClaw-prompt en werkruimtecontext.
3. Voert de CLI uit met een sessie-id (indien ondersteund), zodat de geschiedenis consistent blijft. De gebundelde backend `claude-cli` houdt per OpenClaw-sessie een Claude-stdio-proces actief en verzendt vervolgbeurten via stream-json-stdin.
4. Verwerkt de uitvoer (JSON of platte tekst) en retourneert de uiteindelijke tekst.
5. Slaat sessie-id's per backend persistent op, zodat vervolgbeurten dezelfde CLI-sessie hergebruiken.

## Time-outs en langdurige taken

CLI-backends hebben twee onafhankelijke limieten:

- `agents.defaults.timeoutSeconds` beperkt de volledige agentbeurt. Normale Gateway-beurten nemen de standaardwaarde van 48 uur over; `0` maakt het budget voor de beurt onbeperkt. Een opgeslagen override zoals `600` vervangt die standaardwaarde.
- De watchdog voor ontbrekende CLI-uitvoer stopt een subproces dat stil blijft. Elke backendplugin beheert afzonderlijke profielen voor nieuwe en hervatte sessies, en de watchdog blijft actief, zelfs wanneer het totale budget voor de beurt onbeperkt is.

Verwijder een override voor een korte totale time-out om terug te keren naar de standaardwaarde van 48 uur, of stel een expliciet budget in, zoals 12 uur:

```bash
# Terugkeren naar de standaardwaarde van 48 uur:
openclaw config unset agents.defaults.timeoutSeconds

# Of een expliciete limiet van 12 uur kiezen:
openclaw config set agents.defaults.timeoutSeconds 43200
```

Achtergrondwerk dat binnen een CLI wordt gestart, blijft onderdeel van dat CLI-subproces. Als de bovenliggende beurt zijn totale limiet bereikt, stopt OpenClaw het subproces en de interne achtergrondtaken van de CLI gezamenlijk. Gebruik voor duurzaam langdurig werk een losgekoppelde OpenClaw-[sub-agent](/nl/tools/subagents) of [ACP-agent](/nl/tools/acp-agents); losgekoppelde sub-agents hebben standaard geen uitvoeringstime-out.

De opdracht `openclaw agent` heeft ook een eigen aanvraagdeadline. De fallbackstandaard van 600 seconden geldt voor die opdrachtaanroep, niet voor gewone Gateway-beurten; zie [`openclaw agent`](/nl/cli/agent).

### Bijzonderheden van Claude CLI

De gebundelde backend `claude-cli` geeft de voorkeur aan de systeemeigen skillresolver van Claude Code. Wanneer de huidige Skills-snapshot ten minste één geselecteerde skill met een gematerialiseerd pad bevat, geeft OpenClaw een tijdelijke Claude Code-plugin door via `--plugin-dir` en laat het de dubbele OpenClaw-skillscatalogus weg uit de toegevoegde systeemprompt. Zonder een gematerialiseerde pluginskill behoudt OpenClaw de promptcatalogus als fallback. Overrides voor skillomgevingsvariabelen/API-sleutels blijven tijdens de uitvoering van toepassing op de omgeving van het onderliggende proces.

Claude CLI heeft een eigen niet-interactieve machtigingsmodus; OpenClaw koppelt deze aan het bestaande uitvoeringsbeleid in plaats van Claude-specifieke configuratie toe te voegen. Voor door OpenClaw beheerde live Claude-sessies is het effectieve uitvoeringsbeleid leidend: YOLO (`tools.exec.mode: "full"`) start Claude normaal gesproken met `--permission-mode bypassPermissions`, terwijl een restrictief beleid Claude start met `--permission-mode default`. Gateways die als root worden uitgevoerd, gebruiken ook `default`, omdat Claude Code de bypassmodus voor root weigert. Instellingen per agent in `agents.entries.*.tools.exec` overschrijven voor die agent de globale `tools.exec`. De Anthropic-plugin normaliseert de machtigingsvlaggen van Claude zodat ze overeenkomen met het effectieve beleid en de hostbeperking.

Onder een restrictief beleid vraagt Claude OpenClaw via stdio om toestemming voordat het een van zijn systeemeigen tools of extensietools gebruikt (zijn eigen Bash-, WebFetch- of Claude in Chrome-browsertools). Wanneer de effectieve vraaginstelling voor uitvoering `on-miss` of `always` is, stuurt OpenClaw elk verzoek als interactieve goedkeuring door naar het kanaal van de sessie: **Allow once** staat de afzonderlijke aanroep toe, **Allow always** staat die toolnaam toe voor de rest van de live Claude-sessie (alleen in het geheugen, nooit persistent opgeslagen) en **Deny**, een time-out of een onbereikbare goedkeuringsroute weigeren allemaal de aanroep. Beleid dat nooit om toestemming vraagt, behoudt het oude gedrag: `security: "deny"` weigert elk verzoek, en vragen met `off` en minder dan volledige beveiliging (uitvoeringsmodus `allowlist`) weigert zonder het te vragen.

### Claude-browsertools en aanmelden bij 1Password

Claude Code kan een Chrome-browser besturen via de [Claude in Chrome-extensie](https://code.claude.com/docs/en/chrome), inclusief het automatisch invullen van referenties door [1Password for Claude](/nl/gateway/1password#browser-sign-in-with-1password-for-claude). De gebundelde backend schakelt dit niet in; registreer een [CLI-backendplugin](/nl/plugins/cli-backend-plugins) die `--chrome` toevoegt aan de startargumenten van een backend met het dialect `claude-stream-json`. OpenClaw behoudt een geconfigureerde `--chrome` bij normale uitvoeringen en dwingt altijd `--no-chrome` af bij uitvoeringen met een restrictief toolbeleid, zoals zijvragen. Het Chrome-venster, de extensie en eventuele goedkeuringsprompts van 1Password bevinden zich op de Gateway-host, dus er moet iemand bij die machine aanwezig zijn om het gebruik van referenties goed te keuren.

De backend koppelt ook OpenClaw-niveaus voor `/think` aan de systeemeigen vlag `--effort` van Claude Code: `minimal`/`low` -> `low`, `medium` -> `medium`, en `high`/`xhigh`/`max` worden rechtstreeks doorgegeven. Hierdoor blijven de ondersteunde inspanningsniveaus van Fable 5 gelijk voor Claude CLI met een abonnement en routes met een API-sleutel. `adaptive` verwijdert geconfigureerde vlaggen voor `--effort` en levert geen vervanging, zodat Claude Code de effectieve inspanning bepaalt aan de hand van zijn eigen omgeving, instellingen en modelstandaarden. Andere CLI-backends vereisen dat hun eigenaarplugin een gelijkwaardige argv-mapper declareert voordat `/think` invloed heeft op de gestarte CLI.

Voordat OpenClaw `claude-cli` kan gebruiken, moet Claude Code zelf op dezelfde host zijn aangemeld:

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Voor Docker-installaties moet Claude Code geïnstalleerd zijn en moet je zijn aangemeld binnen de persistente thuismap van de container, niet alleen op de host; zie [Claude CLI-backend in Docker](/nl/install/docker#claude-cli-backend-in-docker).

De Gateway-service moet `claude` kunnen vinden via `PATH`. Registreer voor een niet-standaardpad
een kleine wrapper-backendplugin.

## Sessies

- Als de CLI sessies ondersteunt, stel je `sessionArgs` in met een tijdelijke aanduiding `{sessionId}` (bijvoorbeeld `["--session-id", "{sessionId}"]`).
- Als de CLI een subopdracht voor hervatten met andere vlaggen gebruikt, stel je `resumeArgs` in (vervangt `args` bij het hervatten) en eventueel `resumeOutput` voor hervattingen zonder JSON.
- `sessionMode`:
  - `always`: altijd een sessie-id verzenden (een nieuwe UUID als er geen is opgeslagen).
  - `existing`: alleen een sessie-id verzenden als er eerder een is opgeslagen.
  - `none`: nooit een sessie-id verzenden.
- `claude-cli` gebruikt standaard `liveSession: "claude-stdio"`, `output: "jsonl"` en `input: "stdin"`, zodat vervolgbeurten het actieve Claude-proces hergebruiken zolang dit actief is, ook voor aangepaste configuraties waarin transportvelden ontbreken. Als de Gateway opnieuw wordt gestart of het inactieve proces wordt afgesloten, hervat OpenClaw vanaf de opgeslagen Claude-sessie-id. Opgeslagen sessie-id's worden vóór hervatting gecontroleerd aan de hand van een leesbaar projecttranscript; bij een ontbrekend transcript wordt de koppeling verwijderd (geregistreerd als `reason=transcript-missing`) in plaats van stilzwijgend een nieuwe sessie te starten onder `--resume`.
- Live Claude-sessies gebruiken begrensde beveiligingen voor JSONL-uitvoer: 8 MiB en 20.000 onbewerkte JSONL-regels per beurt.
- Opgeslagen CLI-sessies vormen door de provider beheerde continuïteit. Automatisch resetten is standaard uitgeschakeld; `/reset` en expliciet dagelijks of bij inactiviteit toegepast `session.reset`-beleid beëindigen ze nog steeds.
- Nieuwe CLI-sessies worden normaal gesproken alleen opnieuw gevuld vanuit de Compaction-samenvatting van OpenClaw plus het deel na de Compaction. Om korte sessies te herstellen die vóór Compaction ongeldig zijn geworden, kan een backend zich aanmelden met `reseedFromRawTranscriptWhenUncompacted: true`. Het opnieuw vullen vanuit het onbewerkte transcript blijft begrensd en beperkt tot veilige ongeldigverklaringen, zoals een ontbrekend CLI-transcript, een verweesd uiteinde met toolgebruik, wijzigingen in berichtenbeleid/systeemprompt/cwd/MCP of een nieuwe poging na het verlopen van een sessie; wijzigingen in het authenticatieprofiel of credentialtijdperk vullen de onbewerkte transcriptgeschiedenis nooit opnieuw.

Serialisatie: `serialize: true` houdt uitvoeringen in dezelfde baan op volgorde (de meeste CLI's serialiseren binnen één providerbaan). OpenClaw stopt ook met het hergebruiken van opgeslagen CLI-sessies wanneer de geselecteerde authenticatie-identiteit verandert, waaronder een gewijzigd authenticatieprofiel-id, statische API-sleutel, statisch token of OAuth-accountidentiteit wanneer de CLI er een beschikbaar stelt; alleen rotatie van OAuth-toegangs-/vernieuwingstokens beëindigt de sessie niet. Als een CLI geen stabiele OAuth-account-id heeft, laat OpenClaw die CLI zijn eigen hervattingsmachtigingen afdwingen.

## Fallbackvoorwoord uit claude-cli-sessies

Wanneer een `claude-cli`-poging uitwijkt naar een niet-CLI-kandidaat in [`agents.defaults.model.fallbacks`](/nl/concepts/model-failover), voorziet OpenClaw de volgende poging van een contextpreambule die uit het lokale JSONL-transcript van Claude Code is opgehaald (onder `~/.claude/projects/`, per werkruimte geïndexeerd). Zonder deze aanvangscontext begint de fallbackprovider zonder context, omdat het eigen sessietranscript van OpenClaw leeg is voor `claude-cli`-uitvoeringen.

- De preambule geeft de voorkeur aan de meest recente `/compact`-samenvatting of `compact_boundary`-markering en voegt vervolgens de meest recente beurten na de grens toe tot aan een tekenbudget. Beurten van vóór de grens worden weggelaten omdat de samenvatting ze al vertegenwoordigt.
- Toolblokken worden samengevoegd tot compacte `(tool call: name)`- en `(tool result: …)`-aanwijzingen om het promptbudget realistisch te houden; een te grote samenvatting wordt afgekapt en gelabeld als `(truncated)`.
- Fallbacks van dezelfde provider van `claude-cli` naar `claude-cli` vertrouwen op Claude's eigen `--resume` en slaan de preambule over.
- De aanvangscontext hergebruikt de bestaande validatie van het Claude-sessiebestandspad, zodat willekeurige paden niet kunnen worden gelezen.

## Afbeeldingen

Pluginauteurs declareren ondersteuning voor afbeeldingspaden met `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw schrijft base64-afbeeldingen naar tijdelijke bestanden. Als `imageArg` is ingesteld, worden die paden doorgegeven als CLI-argumenten; zo niet, dan voegt OpenClaw de bestandspaden toe aan de prompt (padinjectie), wat werkt voor CLI's die lokale bestanden automatisch vanuit platte paden laden.

## Invoer en uitvoer

- `output: "text"` (standaard) behandelt stdout als het definitieve antwoord.
- `output: "json"` probeert JSON te parseren en tekst plus een sessie-ID te extraheren.
- `output: "jsonl"` parseert een JSONL-stream en extraheert het definitieve agentbericht plus sessie-ID's wanneer die aanwezig zijn.
- Voor JSON-uitvoer van Gemini CLI leest OpenClaw de antwoordtekst uit `response` en het gebruik uit `stats` wanneer `usage` ontbreekt of leeg is. De gebundelde Gemini CLI-adapter gebruikt `stream-json`.

Invoermodi:

- `input: "arg"` (standaard) geeft de prompt door als het laatste CLI-argument.
- `input: "stdin"` verzendt de prompt via stdin.
- Als de prompt zeer lang is en `maxPromptArgChars` is ingesteld, wordt in plaats daarvan stdin gebruikt.

## Standaardwaarden van de Plugin

Standaardwaarden voor CLI-backends maken deel uit van het Plugin-oppervlak:

- Plugins registreren ze met `api.registerCliBackend(...)`.
- De backend-`id` wordt het providervoorvoegsel in modelverwijzingen.
- Het gedrag van opdracht, argv, omgeving, parser, sessie en watchdog blijft in de Plugincode.
- Backendspecifieke normalisatie blijft eigendom van de Plugin via de optionele `normalizeConfig`-hook.

Anthropic beheert `claude-cli` en Google beheert `google-gemini-cli`. Uitvoeringen van OpenAI Codex-agenten gebruiken de Codex-app-serverharness via `openai/*`; OpenClaw registreert niet langer een gebundelde `codex-cli`-backend.

De gebundelde Anthropic-Plugin registreert voor `claude-cli`:

| Sleutel                | Waarde                                                                                                                                                                                                        |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`             | `claude`                                                                                                                                                                                                      |
| `args`                | `-p --output-format stream-json --include-partial-messages --verbose --setting-sources user --allowedTools mcp__openclaw__* --disallowedTools ScheduleWakeup,CronCreate,Bash(run_in_background:true),Monitor` |
| `output`              | `jsonl`                                                                                                                                                                                                       |
| `input`               | `stdin`                                                                                                                                                                                                       |
| `modelArg`            | `--model`                                                                                                                                                                                                     |
| `sessionArgs`         | `["--session-id", "{sessionId}"]`                                                                                                                                                                             |
| `sessionMode`         | `always`                                                                                                                                                                                                      |
| `imageArg`            | `@`                                                                                                                                                                                                           |
| `imagePathScope`      | `workspace`                                                                                                                                                                                                   |
| `systemPromptFileArg` | `--append-system-prompt-file`                                                                                                                                                                                 |
| `systemPromptMode`    | `append`                                                                                                                                                                                                      |

De gebundelde Google-Plugin registreert voor `google-gemini-cli`:

| Sleutel                   | Waarde                                                                                 |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `command`                 | `gemini`                                                                               |
| `args`                    | `--skip-trust --approval-mode auto_edit --output-format stream-json --prompt {prompt}` |
| `resumeArgs`              | hetzelfde, met `--resume {sessionId}`                                                  |
| `output` / `resumeOutput` | `jsonl`                                                                                |
| `jsonlDialect`            | `gemini-stream-json`                                                                   |
| `imageArg`                | `@`                                                                                    |
| `imagePathScope`          | `workspace`                                                                            |
| `modelArg`                | `--model`                                                                              |
| `sessionMode`             | `existing`                                                                             |
| `sessionIdFields`         | `["session_id", "sessionId"]`                                                          |

Vereiste: de lokale Gemini CLI moet zijn geïnstalleerd en zich op `PATH` bevinden als `gemini` (`brew install gemini-cli` of `npm install -g @google/gemini-cli`).

Opmerkingen over Gemini CLI-uitvoer:

- De standaard `stream-json`-parser leest assistent-`message`-gebeurtenissen, toolgebeurtenissen, het definitieve `result`-gebruik en fatale Gemini-foutgebeurtenissen.
- Het gebruik valt terug op `stats` wanneer `usage` ontbreekt of leeg is; `stats.cached` wordt genormaliseerd naar OpenClaw-`cacheRead`, en als `stats.input` ontbreekt, worden invoertokens afgeleid van `stats.input_tokens - stats.cached`.

## Teksttransformatie-overlays

Plugins die kleine compatibiliteitsshims voor prompts of berichten nodig hebben, kunnen bidirectionele teksttransformaties declareren zonder een provider of CLI-backend te vervangen:

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input` herschrijft de systeemprompt en gebruikersprompt die aan de CLI worden doorgegeven. `output` herschrijft gestreamde assistenttekst en geparseerde definitieve tekst voordat OpenClaw zijn eigen besturingsmarkeringen en kanaalbezorging afhandelt; voor providergebaseerde modelaanroepen herstelt dit ook tekenreekswaarden in gestructureerde toolaanroepargumenten na streamherstel en vóór de tooluitvoering. Ruwe JSON-fragmenten van providers blijven ongewijzigd; consumenten moeten de gestructureerde gedeeltelijke, eind- of resultaatpayload gebruiken.

Stel voor CLI's die providerspecifieke JSONL-gebeurtenissen uitsturen `jsonlDialect` in bij de configuratie van die backend: `claude-stream-json` voor Claude Code-compatibele streams, `gemini-stream-json` voor Gemini CLI-`stream-json`-gebeurtenissen.

## Eigendom van native Compaction

Sommige CLI-backends voeren een agent uit die zijn eigen transcript compacter maakt, zodat OpenClaw zijn beveiligende samenvatter er niet op mag uitvoeren — dat zou de eigen Compaction van de backend tegenwerken en kan de beurt onherstelbaar laten mislukken.

`claude-cli` heeft geen harness-eindpunt (Claude Code voert intern Compaction uit), dus declareert deze `ownsNativeCompaction: true` en retourneert het Compaction-pad van OpenClaw de sessievermelding ongewijzigd. OpenClaw geeft het effectieve contextbudget van de uitvoering door via de gedocumenteerde [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) van Claude Code, zodat native automatische Compaction blijft afgestemd op de geconfigureerde Anthropic-`contextTokens`-limieten. Sessies met een native harness, zoals Codex, blijven in plaats daarvan naar het Compaction-eindpunt van hun harness worden gerouteerd.

```typescript
api.registerCliBackend({ id: "my-cli", ownsNativeCompaction: true /* ... */ });
```

Declareer `ownsNativeCompaction` alleen voor een backend die daadwerkelijk eigenaar is van Compaction: deze moet zijn eigen transcript betrouwbaar begrenzen nabij het contextvenster en een hervatbare sessie opslaan (bijvoorbeeld `--resume` / `--session-id`), anders kan een uitgestelde sessie boven het budget blijven.

## MCP-overlays bundelen

CLI-backends ontvangen OpenClaw-toolaanroepen niet rechtstreeks, maar een backend kan zich met `bundleMcp: true` aanmelden voor een gegenereerde MCP-configuratie-overlay. Huidig gebundeld gedrag:

- `claude-cli`: gegenereerd strikt MCP-configuratiebestand.
- `google-gemini-cli`: gegenereerd Gemini-systeeminstellingenbestand.

Wanneer bundel-MCP is ingeschakeld, doet OpenClaw het volgende:

- start een loopback HTTP MCP-server die Gateway-tools beschikbaar stelt aan het CLI-proces, geverifieerd met een contexttoekenning per uitvoering (`OPENCLAW_MCP_TOKEN`) die alleen actief is voor de huidige uitvoeringspoging;
- koppelt tooltoegang aan de door de Gateway geselecteerde sessie-, account- en kanaalcontext in plaats van headers van het onderliggende proces te vertrouwen;
- laadt ingeschakelde bundel-MCP-servers voor de huidige werkruimte en voegt ze samen met elke bestaande MCP-configuratie- of instellingenstructuur van de backend;
- herschrijft de startconfiguratie met de integratiemodus van de backend die door de beherende Plugin wordt bepaald.

Beperkte uitvoeringen, zoals cron-taken met `toolsAllow`, vereisen een exacte
vertaling die door de backend wordt beheerd. De gebundelde backend `claude-cli` schakelt de
native tools en aanpassingen van Claude op gebruikers-, project- en lokaal niveau uit, waaronder hooks,
plugins, agents, skills en `CLAUDE.md`. Vervolgens stelt deze elke toegestane
OpenClaw-tool beschikbaar via de MCP-server met een tot de toekenning beperkt bereik. Hierdoor blijft het beleid voor het bestandssysteem,
processen, exec, goedkeuringen en de sandbox binnen OpenClaw, in plaats van de
bevoegdheden uit te breiden naar de native tools of aanpassingsprocessen van Claude. Dezelfde MCP-
lijst wordt afgedwongen in de gegenereerde configuratie van Claude en opnieuw door de Gateway bij het
weergeven en uitvoeren van tools. Voordat de toekenning wordt aangemaakt, weigert de kern
backendvertalingen die een MCP-machtiging noemen die buiten de oorspronkelijke lijst met toegestane items valt.
Backends zonder exacte vertaling blijven gesloten bij fouten.

Als er geen MCP-servers zijn ingeschakeld, injecteert OpenClaw nog steeds een strikte configuratie wanneer een backend gebundelde MCP inschakelt, zodat uitvoeringen op de achtergrond geïsoleerd blijven.

Gebundelde MCP-runtimes met sessiebereik worden tijdens een sessie in de cache opgeslagen voor hergebruik en vervolgens na 10 minuten inactiviteit opgeruimd. Eenmalige ingebedde uitvoeringen, zoals authenticatiecontroles, het genereren van slugs en het ophalen van Active Memory, vragen aan het einde van de uitvoering om opschoning, zodat stdio-subprocessen en Streamable HTTP/SSE-streams niet langer blijven bestaan dan de uitvoering.

Voor `claude-cli` wordt een compatibel geselecteerd of geordend OpenClaw OAuth-/tokenprofiel
doorgestuurd naar dat Claude-subproces. Hierdoor zijn profielen per agent bepalend
voor de beurt, terwijl de native hostaanmelding van Claude behouden blijft wanneer er geen compatibel
profiel bestaat.

## Limiet voor opnieuw ingezaaide geschiedenis

Wanneer een nieuwe CLI-sessie wordt ingezaaid vanuit een eerder OpenClaw-transcript (bijvoorbeeld na een nieuwe poging met `session_expired`), wordt het gerenderde blok `<conversation_history>` begrensd om te voorkomen dat prompts voor opnieuw inzaaien explosief groeien. De standaard is 12,288 tekens (ongeveer 3,000 tokens).

Claude CLI-backends schalen deze limiet in plaats daarvan met het bepaalde Claude-contextvenster: grotere contextvensters krijgen een groter fragment van de eerdere geschiedenis, tot een vaste bovengrens; andere CLI-backends behouden de conservatieve standaard. Deze limiet geldt alleen voor het blok met eerdere geschiedenis in de prompt voor opnieuw inzaaien.

## Beperkingen

- OpenClaw injecteert geen toolaanroepen in het CLI-backendprotocol. Backends zien Gateway-tools alleen wanneer ze `bundleMcp: true` inschakelen.
- Streaming is backendspecifiek: sommige backends streamen JSONL, andere bufferen tot het proces wordt afgesloten.
- Gestructureerde uitvoer is afhankelijk van de eigen JSON-indeling van de CLI.

## Probleemoplossing

| Symptoom                  | Oplossing                                                                                                  |
| ------------------------- | ---------------------------------------------------------------------------------------------------------- |
| CLI niet gevonden         | Plaats de CLI in `PATH` van de Gateway-service of werk de geregistreerde opdracht van de beherende Plugin bij. |
| Verkeerde modelnaam       | Werk de toewijzing `modelAliases` van de Plugin bij.                                                   |
| Geen sessiecontinuïteit   | Controleer `sessionArgs` en `sessionMode` van de Plugin.                                         |
| Afbeeldingen genegeerd    | Controleer `imageArg` van de Plugin en de ondersteuning van de CLI voor bestandspaden.              |

## Gerelateerd

- [Gateway-draaiboek](/nl/gateway)
- [Lokale modellen](/nl/gateway/local-models)
