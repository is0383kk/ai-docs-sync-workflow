---
read_when:
    - Fouten opsporen waarom een agent op een bepaalde manier antwoordde, faalde of tools aanriep
    - Een ondersteuningsbundel voor een OpenClaw-sessie exporteren
    - Promptcontext, toolaanroepen, runtimefouten of gebruiksmetadata onderzoeken
    - Trajectregistratie uitschakelen
summary: Geanonimiseerde trajectbundels exporteren voor foutopsporing in een OpenClaw-agentsessie
title: Trajectbundels
x-i18n:
    generated_at: "2026-07-27T06:17:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7fc494732b6239ad4ea58dca3920a47cb7433c680e7566855dd265c986b55e74
    source_path: tools/trajectory.md
    workflow: 16
---

Trajectregistratie is de vluchtrecorder per sessie van OpenClaw. Deze registreert een
gestructureerde tijdlijn voor elke agentuitvoering, waarna `/export-trajectory` de
huidige sessie verpakt in een geredigeerde supportbundel met:

- De prompt, systeemprompt en tools die naar het model zijn verzonden
- Welke transcriptberichten en toolaanroepen tot een antwoord hebben geleid
- Of de uitvoering een time-out kreeg, werd afgebroken, gecompacteerd of een providerfout ondervond
- Welke modellen, plugins, Skills en runtime-instellingen actief waren
- Gebruiks- en promptcachemetadata die de provider heeft geretourneerd

Begin voor een algemeen supportrapport over de Gateway in plaats daarvan met
[`/diagnostics`](/nl/gateway/diagnostics#chat-command); hiermee wordt de
opgeschoonde Gateway-bundel verzameld en kan, voor OpenAI Codex-harnesssessies,
na goedkeuring Codex-feedback naar OpenAI worden verzonden. Gebruik `/export-trajectory`
wanneer je de gedetailleerde tijdlijn per sessie van prompts, tools en het transcript nodig hebt.

## Snel aan de slag

Verzend in de actieve sessie (alias `/trajectory`):

```text
/export-trajectory
```

OpenClaw schrijft de bundel naar de werkruimte:

```text
.openclaw/trajectory-exports/openclaw-trajectory-<session>-<timestamp>/
```

Geef een relatieve naam voor de uitvoermap door om deze te overschrijven:

```text
/export-trajectory bug-1234
```

De naam wordt binnen `.openclaw/trajectory-exports/` omgezet. Absolute paden en
`~`-paden worden geweigerd.

Trajectbundels kunnen prompts, modelberichten, toolschema's, toolresultaten,
runtimegebeurtenissen en lokale paden bevatten. Daarom verloopt de chatopdracht
altijd via uitvoeringsgoedkeuring. Keur de export één keer goed wanneer je de
bundel wilt maken; gebruik niet alles toestaan. In groepschats stuurt OpenClaw
de goedkeuringsprompt en het exportresultaat privé naar de eigenaar, in plaats
van trajectdetails in de gedeelde ruimte te plaatsen.

Voer voor lokale inspectie of supportworkflows de onderliggende CLI-opdracht
rechtstreeks uit:

```bash
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --workspace .
```

Andere vlaggen: `--output <path>` (mapnaam binnen
`.openclaw/trajectory-exports`), `--store <path>` (overschrijving van de sessieopslag),
`--agent <id>` (agent-id voor opslagomzetting), `--json` (gestructureerde uitvoer).

## Toegang

Trajectexport is een eigenaarsopdracht. De afzender moet slagen voor de normale
autorisatiecontroles voor opdrachten en voor de eigenaarscontrole van het kanaal.

## Wat wordt geregistreerd

Trajectregistratie is standaard ingeschakeld voor agentuitvoeringen van OpenClaw.

Runtimegebeurtenissen omvatten:

- `session.started`
- `trace.metadata`
- `context.compiled`
- `prompt.submitted`
- `model.fallback_step`, inclusief het bronmodel, het volgende model, de reden/details van de fout, de positie in de keten en of de keten is doorgegaan, geslaagd of uitgeput
- `model.completed`
- `trace.artifacts`
- `session.ended`

Transcriptgebeurtenissen worden gereconstrueerd vanuit de actieve sessietak:
gebruikersberichten, assistentberichten, toolaanroepen, toolresultaten,
Compaction-bewerkingen, modelwijzigingen, labels en aangepaste sessie-items.

Gebeurtenissen worden als JSON Lines geschreven met deze schemamarkering:

```json
{
  "traceSchema": "openclaw-trajectory",
  "schemaVersion": 1
}
```

## Bundelbestanden

| Bestand               | Inhoud                                                                                               |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| `manifest.json`       | Bundelschema, bronbestanden, aantallen gebeurtenissen en lijst met gegenereerde bestanden            |
| `events.jsonl`        | Geordende tijdlijn van runtime en transcript                                                         |
| `session-branch.json` | Geredigeerde actieve transcripttak en sessiekoptekst                                                 |
| `metadata.json`       | OpenClaw-versie, besturingssysteem/runtime, model, configuratiemomentopname, plugins, Skills en promptmetadata |
| `artifacts.json`      | Eindstatus, fouten, gebruik, promptcache, aantal Compaction-bewerkingen, assistenttekst en toolmetadata |
| `prompts.json`        | Ingediende prompts en geselecteerde details over het opbouwen van prompts                            |
| `system-prompt.txt`   | Meest recente gecompileerde systeemprompt, indien geregistreerd                                      |
| `tools.json`          | Tooldefinities die naar het model zijn verzonden, indien geregistreerd                               |

`manifest.json` vermeldt de bestanden die in een bepaalde bundel aanwezig
zijn; sommige bestanden worden weggelaten wanneer de sessie de bijbehorende
runtimegegevens niet heeft geregistreerd.

## Opslag van registraties

Runtimegebeurtenissen van trajecten worden samen met de sessie opgeslagen in de
SQLite-database per agent. Bij het exporteren van een traject wordt een
geredigeerde JSONL-supportbundel aangemaakt; de actieve runtimeregistratie is
geen JSONL-nevenbestand naast de sessie.

Verouderde `.trajectory.jsonl`- en `.trajectory-path.json`-bestanden kunnen nog
voorkomen uit oudere releases of expliciete exports naar verouderde bestanden.
Sessieonderhoud behandelt die bestanden als opschoondoelen; actieve registratie
schrijft databaserijen.

## Registratie uitschakelen

```bash
export OPENCLAW_TRAJECTORY=0
```

Hiermee wordt de runtimetrajectregistratie uitgeschakeld voordat OpenClaw wordt
gestart. `/export-trajectory` kan de transcripttak nog steeds exporteren, maar
gegevens die alleen tijdens runtime beschikbaar zijn, zoals gecompileerde
context, providerartefacten en promptmetadata, kunnen ontbreken.

## Time-out voor wegschrijven aanpassen

OpenClaw schrijft runtimetrajectrijen weg tijdens het opruimen van de agent. De
standaardtime-out voor opruimen is 10,000 ms. Stel op langzame schijven of bij
grote opslagplaatsen `OPENCLAW_TRAJECTORY_FLUSH_TIMEOUT_MS` in voordat OpenClaw wordt gestart:

```bash
export OPENCLAW_TRAJECTORY_FLUSH_TIMEOUT_MS=30000
```

Dit bepaalt wanneer OpenClaw een `openclaw-trajectory-flush`-time-out registreert en
doorgaat; het wijzigt de maximale trajectgroottes niet. Stel
`OPENCLAW_AGENT_CLEANUP_TIMEOUT_MS` in om alle opruimstappen van agents aan te passen waarvoor
geen expliciete time-out wordt doorgegeven.

## Privacy en limieten

Trajectbundels zijn bedoeld voor support en foutopsporing, niet voor openbare
publicatie. OpenClaw redigeert gevoelige waarden voordat exportbestanden worden
geschreven:

- aanmeldgegevens en bekende payloadvelden die op geheimen lijken
- afbeeldingsgegevens
- lokale statuspaden
- werkruimtepaden, vervangen door `$WORKSPACE_DIR`
- paden naar de thuismap, waar gedetecteerd

De exportfunctie begrenst ook de invoergrootte:

- runtimeregistratie: de actieve registratie is een voortschrijdend venster met een maximum van 10 MiB, waarbij de oudste gebeurtenissen worden verwijderd om ruimte te maken voor nieuwe; bij export worden bestaande verouderde runtime-nevenbestanden tot 50 MiB geaccepteerd
- sessiebestanden: 50 MiB
- runtimegebeurtenissen per export: 200,000
- totaal aantal geëxporteerde gebeurtenissen: 250,000
- afzonderlijke regels met runtimegebeurtenissen worden boven 256 KiB afgekapt

Controleer bundels voordat je ze buiten je team deelt. Redigering gebeurt naar
beste vermogen en kan niet elk toepassingsspecifiek geheim herkennen.

## Problemen oplossen

Als de export geen runtimegebeurtenissen bevat:

- controleer of OpenClaw zonder `OPENCLAW_TRAJECTORY=0` is gestart
- voer nog een bericht uit in de sessie en exporteer vervolgens opnieuw
- inspecteer `manifest.json` op `runtimeEventCount`

Als de opdracht het uitvoerpad weigert:

- gebruik een relatieve naam zoals `bug-1234`
- geef `/tmp/...` of `~/...` niet door
- houd de export binnen `.openclaw/trajectory-exports/`

Als de export mislukt met een groottefout, heeft de sessie of het nevenbestand
de bovenstaande veiligheidslimieten voor export overschreden. Start een nieuwe
sessie of exporteer een kleinere reproductie.

## Gerelateerd

- [Verschillen](/nl/tools/diffs)
- [Sessiebeheer](/nl/concepts/session)
- [Uitvoeringstool](/nl/tools/exec)
