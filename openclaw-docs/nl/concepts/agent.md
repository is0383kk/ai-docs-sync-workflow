---
read_when:
    - Agentruntime, werkruimte-initialisatie of sessiegedrag wijzigen
summary: Agentruntime, werkruimtecontract en sessie-initialisatie
title: Agentruntime
x-i18n:
    generated_at: "2026-07-27T06:10:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4d3dd9c0c65e4ccd791a2a6131f1b7457c8cfee6da71502d93c355280e094390
    source_path: concepts/agent.md
    workflow: 16
---

OpenClaw levert één **ingebouwde agentruntime**: een ingebouwde agentlus, toolkoppeling en promptopbouw, los van het delegeren van beurten aan een extern harnessproces. Elke geconfigureerde agent (zie [Routering met meerdere agents](/nl/concepts/multi-agent) voor het uitvoeren van meerdere agents) heeft een eigen werkruimte, bootstrapbestanden en sessieopslag. Deze pagina behandelt het runtimecontract: wat de werkruimte moet bevatten, welke bestanden worden geïnjecteerd en hoe sessies daarmee worden opgestart.

## Werkruimte (vereist)

Elke agent gebruikt één werkruimtemap (`agents.defaults.workspace`, of
`agents.entries.*.workspace` per agent) als zijn **enige** werkmap (`cwd`)
voor tools en context.

Aanbevolen: gebruik `openclaw setup` om `~/.openclaw/openclaw.json` aan te maken als deze ontbreekt en de werkruimtebestanden te initialiseren.

Volledige indeling van de werkruimte + back-uphandleiding: [Agentwerkruimte](/nl/concepts/agent-workspace)

Als `agents.defaults.sandbox` is ingeschakeld, kunnen niet-hoofdsessies dit overschrijven met
werkruimten per sessie onder `agents.defaults.sandbox.workspaceRoot` (zie
[Gateway-configuratie](/nl/gateway/configuration)).

## Bootstrapbestanden (geïnjecteerd)

In de werkruimte verwacht OpenClaw deze door de gebruiker bewerkbare bestanden:

| Bestand        | Doel                                                 |
| -------------- | ---------------------------------------------------- |
| `AGENTS.md`    | Gebruiksinstructies + "geheugen"                     |
| `SOUL.md`      | Persona, grenzen, toon                               |
| `TOOLS.md`     | Door de gebruiker beheerde toolnotities en conventies |
| `IDENTITY.md`  | Naam/sfeer/emoji van de agent                        |
| `USER.md`      | Gebruikersprofiel + voorkeursaanspreekvorm           |
| `HEARTBEAT.md` | Heartbeat-specifieke instructies                     |
| `BOOTSTRAP.md` | Eenmalig ritueel bij de eerste uitvoering (na voltooiing verwijderd) |
| `MEMORY.md`    | Hoofdbestand voor langetermijngeheugen, indien aanwezig |

Tijdens de eerste beurt van een nieuwe sessie injecteert OpenClaw de inhoud van deze bestanden in de Projectcontext van de systeemprompt. `MEMORY.md` wordt alleen geïnjecteerd wanneer het in de hoofdmap van de werkruimte bestaat.

Lege bestanden worden overgeslagen. Grote bestanden worden ingekort en afgekapt met een markering, zodat prompts beknopt blijven (lees het bestand voor de volledige inhoud). Voor een ontbrekend bestand (behalve `MEMORY.md`) wordt in plaats daarvan één markeringsregel voor een "ontbrekend bestand" geïnjecteerd; `openclaw setup` maakt hiervoor een veilige standaardsjabloon.

`BOOTSTRAP.md` wordt alleen aangemaakt voor een **volledig nieuwe werkruimte** (waarin geen andere bootstrapbestanden aanwezig zijn). Zolang dit bestand in behandeling is, houdt OpenClaw het in de Projectcontext en voegt het bootstrapbegeleiding voor het eerste ritueel toe aan de systeemprompt, in plaats van het naar het gebruikersbericht te kopiëren. Als je het na voltooiing van het ritueel verwijdert, wordt het bij latere herstarts niet opnieuw aangemaakt.

Nadat een werkruimte is waargenomen, slaat OpenClaw de instellingsstatus en
attestatie ervan op in de gedeelde SQLite-database op
`~/.openclaw/state/openclaw.sqlite`. Als een onlangs geattesteerde werkruimte
verdwijnt of wordt gewist, weigert het opstartproces `BOOTSTRAP.md` stilzwijgend opnieuw te vullen;
herstel de werkruimte of voer een volledige onboardingreset uit, zodat de werkruimte en
de databasestatus ervan samen worden gewist.

Oudere releases gebruikten JSON-bestanden voor werkruimten en `.attested`-sidecarbestanden. De runtime leest
deze bestanden niet. Voer `openclaw doctor --fix` uit om ze te valideren, hun
status in SQLite te importeren en elke bron te verwijderen nadat de geïmporteerde rijen zijn geverifieerd.

Stel het volgende in om het aanmaken van bootstrapbestanden volledig uit te schakelen (voor vooraf gevulde werkruimten):

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## Ingebouwde tools

Kerntools (lezen/uitvoeren/bewerken/schrijven en gerelateerde systeemtools) zijn altijd beschikbaar,
onder voorbehoud van het toolbeleid. `apply_patch` is standaard ingeschakeld voor OpenAI-modellen en wordt beheerst door
`tools.exec.applyPatch` (`enabled`, `workspaceOnly`, `allowModels`). `TOOLS.md` bepaalt **niet** welke tools bestaan; het is
een richtlijn voor hoe _je_ wilt dat ze worden gebruikt.

## Skills

OpenClaw laadt Skills vanaf deze locaties (hoogste prioriteit eerst):

- Werkruimte: `<workspace>/skills`
- Agent-Skills van het project: `<workspace>/.agents/skills`
- Persoonlijke agent-Skills: `~/.agents/skills`
- Beheerd/lokaal: `~/.openclaw/skills`
- Gebundeld (meegeleverd met de installatie)
- Extra mappen met Skills: `skills.load.extraDirs`

Hoofdmappen van Skills kunnen gegroepeerde mappen bevatten, zoals
`<workspace>/skills/personal/foo/SKILL.md`; de Skill wordt nog steeds beschikbaar gesteld onder de
platte frontmatter-naam, bijvoorbeeld `foo`.

Skills kunnen worden beheerst door configuratie/omgevingsvariabelen (zie `skills` in [Gateway-configuratie](/nl/gateway/configuration)).

## Runtimegrenzen

De ingebouwde agentruntime is eigendom van OpenClaw: modeldetectie, toolkoppeling,
promptopbouw, sessiebeheer en kanaalbezorging delen één geïntegreerd
runtimeoppervlak.

## Sessies

Sessierijen worden opgeslagen in de SQLite-database per agent:

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

JSONL-transcriptbestanden kunnen nog steeds onder
`~/.openclaw/agents/<agentId>/sessions/` staan als invoer voor verouderde migraties, verwijderde of
geresette archieven, imports, exports en ondersteuningsartefacten. Actieve agentgeschiedenis wordt
samen met de sessierijen in SQLite opgeslagen. De sessie-ID is stabiel en wordt door
OpenClaw gekozen. OpenClaw leest geen sessiemappen van andere tools.

## Bijsturen tijdens streamen

Binnenkomende prompts die tijdens een uitvoering arriveren, worden standaard naar de huidige uitvoering gestuurd.
Bijsturing wordt geleverd **nadat de huidige assistentbeurt klaar is met het uitvoeren van de
toolaanroepen**, vóór de volgende LLM-aanroep, en slaat resterende toolaanroepen
uit het huidige assistentbericht niet langer over.

`/queue steer` is het standaardgedrag tijdens een actieve uitvoering. `/queue followup` en
`/queue collect` laten berichten wachten op een latere beurt in plaats van ze bij te sturen.
`/queue interrupt` breekt in plaats daarvan de actieve uitvoering af. Zie [Wachtrij](/nl/concepts/queue)
en [Bijsturingswachtrij](/nl/concepts/queue-steering) voor het gedrag van wachtrijen en grenzen.

Blokstreaming verzendt voltooide assistentblokken zodra ze gereed zijn; dit is
**standaard uitgeschakeld** (`agents.defaults.blockStreamingDefault: "off"`).
Stel de grens af via `agents.defaults.blockStreamingBreak` (`text_end` tegenover `message_end`; standaard `text_end`).
Beheer het opdelen in zachte blokken met `agents.defaults.blockStreamingChunk` (standaard
800-1200 tekens; geeft de voorkeur aan alinea-einden, daarna regeleinden; zinnen als laatste).
Voeg gestreamde fragmenten samen met `agents.defaults.blockStreamingCoalesce` om
spam van afzonderlijke regels te verminderen (samenvoeging op basis van inactiviteit vóór verzending). Voor andere kanalen dan Telegram is
expliciet `*.streaming.block.enabled: true` vereist om blokantwoorden in te schakelen (QQ Bot
streamt blokantwoorden juist, tenzij `channels.qqbot.streaming.mode` `"off"` is).
Uitgebreide toolsamenvattingen worden bij het starten van de tool gegenereerd (zonder debounce); de Control UI
streamt tooluitvoer via agentgebeurtenissen wanneer beschikbaar.
Meer informatie: [Streamen + opdelen](/nl/concepts/streaming).

## Modelverwijzingen

Modelverwijzingen in de configuratie (bijvoorbeeld `agents.defaults.model` en `agents.defaults.models`) worden geparseerd door ze te splitsen op de **eerste** `/`.

- Gebruik `provider/model` bij het configureren van modellen.
- Als de model-ID zelf `/` bevat (OpenRouter-stijl), neem dan het providerprefix op (voorbeeld: `openrouter/moonshotai/kimi-k2`).
- Als je de provider weglaat, probeert OpenClaw eerst een alias, daarna een unieke
  overeenkomst met een geconfigureerde provider voor die exacte model-ID, en valt het pas daarna terug
  op de geconfigureerde standaardprovider. Als die provider het
  geconfigureerde standaardmodel niet langer aanbiedt, valt OpenClaw terug op het eerste geconfigureerde
  provider/model in plaats van een verouderde standaard van een verwijderde provider te tonen.

## Configuratie (minimaal)

Stel minimaal het volgende in:

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom` (sterk aanbevolen)

## Gerelateerd

- [Agentwerkruimte](/nl/concepts/agent-workspace)
- [Routering met meerdere agents](/nl/concepts/multi-agent)
- [Sessiebeheer](/nl/concepts/session)
- [Groepschats](/nl/channels/group-messages)
