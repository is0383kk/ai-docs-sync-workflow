---
read_when:
    - Je wilt gebeurtenisgestuurde automatisering voor /new, /reset, /stop en levenscyclusgebeurtenissen van agents
    - Je wilt hooks bouwen, installeren of debuggen
summary: 'Hooks: gebeurtenisgestuurde automatisering voor opdrachten en levenscyclusgebeurtenissen'
title: Hooks
x-i18n:
    generated_at: "2026-07-27T05:00:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 039a55cca60e0005d7b9c4d950a86aceb6e7c29d5768108b34011bfc21c85be6
    source_path: automation/hooks.md
    workflow: 16
---

Hooks zijn kleine scripts die binnen de Gateway worden uitgevoerd wanneer agentgebeurtenissen plaatsvinden: opdrachten zoals `/new`, `/reset`, `/stop`, sessiecompactie, de levenscyclus van de Gateway en de berichtenstroom. Ze worden in mappen gedetecteerd en beheerd met `openclaw hooks`. De Gateway laadt interne hooks pas nadat je hooks hebt ingeschakeld of ten minste één hookvermelding, hookpakket, verouderde handler of extra hookmap hebt geconfigureerd.

OpenClaw heeft twee soorten hooks:

- **Interne hooks** (deze pagina): worden binnen de Gateway uitgevoerd wanneer agentgebeurtenissen plaatsvinden.
- **Webhooks**: externe HTTP-eindpunten waarmee andere systemen werk in OpenClaw kunnen activeren. Zie [Webhooks](/nl/automation/cron-jobs#webhooks).

Hooks kunnen ook in plugins worden gebundeld. `openclaw hooks list` toont zowel zelfstandige hooks als door plugins beheerde hooks (weergegeven als `plugin:<id>`).

## Kies het juiste uitbreidingspunt

OpenClaw heeft verschillende uitbreidingspunten die op elkaar lijken, maar verschillende problemen oplossen:

| Als je het volgende wilt...                                                                                                     | Gebruik...                                | Waarom                                                                                           |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| Een momentopname opslaan bij `/new`, `/reset` vastleggen, een externe API aanroepen na `message:sent` of algemene automatisering voor operators toevoegen | Interne hooks (`HOOK.md`, deze pagina) | Bestandsgebaseerde hooks zijn bedoeld voor door operators beheerde neveneffecten en automatisering van opdrachten en de levenscyclus |
| Prompts herschrijven, tools blokkeren, uitgaande berichten annuleren of geordende middleware/beleidsregels toevoegen                              | Getypeerde pluginhooks via `api.on(...)`  | Getypeerde hooks hebben expliciete contracten, prioriteiten, samenvoegingsregels en semantiek voor blokkeren en annuleren      |
| Alleen telemetrie exporteren of observeerbaarheid toevoegen                                                                            | Diagnostische gebeurtenissen                     | Observeerbaarheid gebruikt een afzonderlijke gebeurtenisbus en is geen uitbreidingspunt voor beleid                              |

Gebruik interne hooks voor automatisering die zich gedraagt als een kleine geïnstalleerde integratie. Gebruik getypeerde pluginhooks wanneer je controle over de runtimelevenscyclus nodig hebt.

## Snel aan de slag

```bash
# Beschikbare hooks weergeven
openclaw hooks list

# Een hook inschakelen
openclaw hooks enable session-memory

# Hookstatus controleren
openclaw hooks check

# Gedetailleerde informatie ophalen
openclaw hooks info session-memory
```

## Gebeurtenistypen

Hooks abonneren zich op een specifieke sleutel uit deze tabel, of op alleen een familienaam
(`command`, `session`, `agent`, `gateway`, `message`) om elke actie
in die familie te ontvangen. De kern van OpenClaw zendt niets anders uit, dus elke andere naam is vrijwel
altijd een typefout waardoor de hook stilzwijgend inactief blijft (alleen een plugin die een
aangepaste gebeurtenis uitzendt, kan deze activeren). De hooklader registreert een waarschuwing voor zulke namen
(bijvoorbeeld `command:nwe`) en `openclaw hooks info <name>` markeert ze, zodat een
hook die nooit wordt uitgevoerd kan worden gediagnosticeerd.

| Gebeurtenis                    | Wanneer deze plaatsvindt                                              |
| ------------------------ | ---------------------------------------------------------- |
| `command:new`            | De opdracht `/new` wordt gegeven                                      |
| `command:reset`          | De opdracht `/reset` wordt gegeven                                    |
| `command:stop`           | De opdracht `/stop` wordt gegeven                                     |
| `command`                | Elke opdrachtgebeurtenis (algemene listener)                       |
| `session:compact:before` | Voordat Compaction de geschiedenis samenvat                       |
| `session:compact:after`  | Nadat Compaction is voltooid                                 |
| `session:patch`          | Wanneer sessie-eigenschappen worden gewijzigd                       |
| `agent:bootstrap`        | Voordat bootstrapbestanden van de werkruimte worden geïnjecteerd              |
| `gateway:startup`        | Nadat kanalen zijn gestart en hooks zijn geladen                  |
| `gateway:shutdown`       | Wanneer het afsluiten van de Gateway begint                               |
| `gateway:pre-restart`    | Vóór een verwachte herstart van de Gateway                         |
| `message:received`       | Inkomend bericht van elk kanaal                           |
| `message:transcribed`    | Nadat de audiotranscriptie is voltooid                        |
| `message:preprocessed`   | Nadat de voorverwerking van media en links is voltooid of overgeslagen |
| `message:sent`           | Poging om een uitgaand bericht te verzenden (`context.success` bevat het resultaat) |

## Hooks schrijven

### Hookstructuur

Elke hook is een map met twee bestanden:

```text
my-hook/
├── HOOK.md          # Metagegevens + documentatie
└── handler.ts       # Handlerimplementatie
```

Het handlerbestand kan `handler.ts`, `handler.js`, `index.ts` of `index.js` zijn.

### Indeling van HOOK.md

```markdown
---
name: my-hook
description: "Korte beschrijving van wat deze hook doet"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# Mijn hook

Hier komt de gedetailleerde documentatie.
```

**Metagegevensvelden** (`metadata.openclaw`):

| Veld      | Beschrijving                                          |
| ---------- | ---------------------------------------------------- |
| `emoji`    | Weergave-emoji voor de CLI                                |
| `events`   | Reeks gebeurtenissen waarop wordt geluisterd                        |
| `export`   | Te gebruiken benoemde export (standaard `"default"`)        |
| `os`       | Vereiste platforms (bijvoorbeeld `["darwin", "linux"]`)     |
| `requires` | Vereiste paden voor `bins`, `anyBins`, `env` of `config` |
| `always`   | Geschiktheidscontroles omzeilen (booleaans)                  |
| `hookKey`  | Configuratiesleutel overschrijven (standaard de hooknaam)      |
| `homepage` | Documentatie-URL die door `openclaw hooks info` wordt weergegeven              |
| `install`  | Installatiemethoden                                 |

### Handlerimplementatie

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] Nieuwe opdracht geactiveerd`);
  // Jouw logica komt hier

  // Stuur desgewenst een antwoord op oppervlakken die antwoorden ondersteunen
  event.messages.push("Hook uitgevoerd!");
};

export default handler;
```

Elke gebeurtenis bevat: `type`, `action`, `sessionKey`, `timestamp`, `messages` en `context` (gebeurtenisspecifieke gegevens). Contexten van getypeerde pluginhooks voor agent- en toolhooks kunnen ook `trace` bevatten, een alleen-lezen, W3C-compatibele diagnostische traceercontext die plugins kunnen doorgeven aan gestructureerde logboeken voor OTEL-correlatie.

Tekenreeksen die aan `event.messages` worden toegevoegd, worden alleen teruggestuurd naar de chat voor
`command:new` en `command:reset` (gerouteerd als antwoord op het oorspronkelijke
gesprek) en voor `session:compact:before` / `session:compact:after`
(verzonden als statusmeldingen over Compaction). Alle andere gebeurtenissen, waaronder
`command:stop`, `message:*`, `agent:bootstrap`, `session:patch` en
`gateway:*`, negeren toegevoegde berichten.

### Belangrijkste onderdelen van de gebeurteniscontext

**Opdrachtgebeurtenissen** (`command:new`, `command:reset`): `context.sessionEntry`, `context.previousSessionEntry`, `context.commandSource`, `context.senderId`, `context.workspaceDir`, `context.cfg`.

**Opdrachtgebeurtenissen** (`command:stop`): `context.sessionEntry`, `context.sessionId`, `context.commandSource`, `context.senderId`.

**Berichtgebeurtenissen** (`message:received`): `context.from`, `context.content`, `context.channelId`, `context.media` (geordende, voorbereide feiten over bijlagen), `context.originalMedia` plus `context.mediaStagingPending` wanneer externe media nog niet lokaal zijn voorbereid, en `context.metadata` (providerspecifieke gegevens, waaronder `senderId`, `senderName`, `guildId`). `context.content` geeft voor berichten die op opdrachten lijken de voorkeur aan een niet-lege opdrachttekst en valt vervolgens terug op de onbewerkte inkomende tekst en de algemene tekst; het bevat geen verrijking die alleen voor agents beschikbaar is, zoals threadgeschiedenis of linksamenvattingen. Verouderde media-aliassen binnen `metadata` zijn afgeschaft.

**Berichtgebeurtenissen** (`message:sent`): `context.to`, `context.content`, `context.success`, `context.channelId`, plus `context.error` wanneer het verzenden is mislukt.

**Berichtgebeurtenissen** (`message:transcribed`): `context.transcript`, `context.from`, `context.channelId` en `context.media`. `context.mediaPath` en `context.mediaType` blijven verouderde aliassen voor het eerste feit.

**Berichtgebeurtenissen** (`message:preprocessed`): `context.bodyForAgent` (uiteindelijke verrijkte tekst), `context.from`, `context.channelId`.

**Bootstrapgebeurtenissen** (`agent:bootstrap`): `context.bootstrapFiles` (wijzigbare reeks), `context.agentId`.

**Gebeurtenissen voor sessiewijzigingen** (`session:patch`): `context.sessionEntry`, `context.patch` (alleen gewijzigde velden), `context.cfg`. Alleen geprivilegieerde clients kunnen wijzigingsgebeurtenissen activeren; de context is een kloon, zodat handlers het actieve sessie-item niet kunnen wijzigen.

**Compaction-gebeurtenissen**: `session:compact:before` bevat `messageCount`, `tokenCount`. `session:compact:after` voegt `compactedCount`, `summaryLength`, `tokensBefore`, `tokensAfter` toe.

`command:stop` neemt waar dat de gebruiker `/stop` geeft; dit betreft annulering/de levenscyclus van de opdracht
en is geen poort voor het voltooien van de agent. Plugins die een
natuurlijk definitief antwoord moeten inspecteren en de agent om nog een verwerkingsronde willen vragen, moeten in plaats daarvan de getypeerde
pluginhook `before_agent_finalize` gebruiken. Zie [Pluginhooks](/nl/plugins/hooks).

**Gebeurtenissen in de levenscyclus van de Gateway**: `gateway:shutdown` bevat `reason` en `restartExpectedMs` en wordt geactiveerd wanneer het afsluiten van de Gateway begint. `gateway:pre-restart` bevat dezelfde context, maar wordt alleen geactiveerd wanneer het afsluiten deel uitmaakt van een verwachte herstart en een eindige waarde voor `restartExpectedMs` wordt opgegeven. Tijdens het afsluiten wordt op elke levenscyclushook naar beste vermogen en gedurende een begrensde periode gewacht, zodat het afsluiten doorgaat als een handler vastloopt. Het standaardwachtbudget is 5 seconden voor `gateway:shutdown` en 10 seconden voor `gateway:pre-restart`.

Gebruik `gateway:pre-restart` voor korte herstartmeldingen terwijl kanalen nog beschikbaar zijn:

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

export default async function handler(event) {
  if (event.type !== "gateway" || event.action !== "pre-restart") {
    return;
  }

  const restartInSeconds = Math.ceil(event.context.restartExpectedMs / 1000);
  await execFileAsync("openclaw", [
    "system",
    "event",
    "--mode",
    "now",
    "--text",
    `Gateway wordt over ongeveer ${restartInSeconds}s opnieuw gestart (${event.context.reason}). Maak nu een controlepunt.`,
  ]);
}
```

Tussen de gebeurtenis `gateway:shutdown` (of `gateway:pre-restart`) en de rest van de afsluitvolgorde activeert de Gateway ook een getypeerde pluginhook `session_end` voor elke sessie die nog actief was toen het proces stopte. De `reason` van de gebeurtenis is `shutdown` voor een gewone stop via SIGTERM/SIGINT en `restart` wanneer het sluiten is gepland als onderdeel van een verwachte herstart. Deze afhandeling is begrensd, zodat een trage `session_end`-handler het afsluiten van het proces niet kan blokkeren. Sessies die al zijn voltooid via vervangen / opnieuw instellen / verwijderen / Compaction worden overgeslagen om dubbele activering te voorkomen.

## Hookdetectie

Hooks worden uit vier bronnen gedetecteerd:

1. **Gebundelde hooks**: meegeleverd met OpenClaw
2. **Pluginhooks**: gebundeld in geïnstalleerde plugins; kunnen gebundelde hooks met dezelfde naam overschrijven
3. **Beheerde hooks**: `~/.openclaw/hooks/` (door de gebruiker geïnstalleerd, gedeeld tussen werkruimten); kunnen gebundelde hooks en pluginhooks overschrijven. Extra mappen uit `hooks.internal.load.extraDirs` hebben dezelfde prioriteit.
4. **Werkruimtehooks**: `<workspace>/hooks/` (per agent, standaard uitgeschakeld totdat ze expliciet worden ingeschakeld)

Werkruimtehooks kunnen nieuwe hooknamen toevoegen, maar kunnen geen gebundelde, beheerde of door plugins geleverde hooks met dezelfde naam overschrijven.

De Gateway slaat bij het opstarten de detectie van interne hooks over totdat interne hooks zijn geconfigureerd. Schakel een gebundelde of beheerde hook in met `openclaw hooks enable <name>`, installeer een hookpakket of stel `hooks.internal.enabled=true` in om hiervoor te kiezen. Wanneer je één benoemde hook inschakelt, laadt de Gateway alleen de handler van die hook; `hooks.internal.enabled=true`, extra hookmappen en verouderde handlers schakelen brede detectie in.

### Hookpakketten

Hookpakketten zijn npm-pakketten die hooks exporteren via `openclaw.hooks` in `package.json`. Installeer ze met:

```bash
openclaw plugins install <path-or-spec>
```

Npm-specificaties zijn beperkt tot het register (pakketnaam + optionele exacte versie of dist-tag). Git-/URL-/bestandsspecificaties en semver-bereiken worden geweigerd. De oudere opdrachten `openclaw hooks install` en `openclaw hooks update` zijn verouderde aliassen voor `openclaw plugins install` / `openclaw plugins update`.

## Gebundelde hooks

| Hook                  | Gebeurtenissen                                    | Functie                                                         |
| --------------------- | ------------------------------------------------- | --------------------------------------------------------------- |
| session-memory        | `command:new`, `command:reset`                    | Slaat sessiecontext op in `<workspace>/memory/`                 |
| bootstrap-extra-files | `agent:bootstrap`                                 | Injecteert aanvullende bootstrapbestanden uit globpatronen      |
| command-logger        | `command`                                         | Logt alle opdrachten naar `~/.openclaw/logs/commands.log`           |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | Stuurt zichtbare chatmeldingen wanneer sessiecompactie begint/eindigt |
| boot-md               | `gateway:startup`                                 | Voert `BOOT.md` uit wanneer de Gateway start                         |

Schakel een gebundelde hook in:

```bash
openclaw hooks enable <hook-name>
```

<a id="session-memory"></a>

### Details van session-memory

Extraheert de laatste berichten van de gebruiker/assistent (standaard 15, configureerbaar met `hooks.internal.entries.session-memory.messages`) en slaat ze op in `<workspace>/memory/YYYY-MM-DD-HHMM.md` met de lokale datum van de host. Het vastleggen van het geheugen wordt op de achtergrond uitgevoerd, zodat bevestigingen van `/new` en `/reset` niet worden vertraagd door het lezen van transcripten of het optioneel genereren van slugs. Stel `hooks.internal.entries.session-memory.llmSlug: true` in om beschrijvende slugs voor bestandsnamen te genereren en stel eventueel `hooks.internal.entries.session-memory.model` in op een geconfigureerde alias zoals `sonnet`, een kale model-ID bij de standaardprovider van de agent of een `provider/model`-verwijzing. Voor het genereren van slugs wordt het standaardmodel van de agent gebruikt wanneer `model` is weggelaten; als dit niet beschikbaar is, wordt teruggevallen op slugs met tijdstempels. Vereist dat `workspace.dir` is geconfigureerd.

<a id="bootstrap-extra-files"></a>

### Configuratie van bootstrap-extra-files

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

`patterns` en `files` worden geaccepteerd als aliassen van `paths`. Paden worden relatief ten opzichte van de werkruimte opgelost en moeten daarbinnen blijven. Alleen herkende bootstrapbasisnamen worden geladen (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`, `MEMORY.md`).

<a id="command-logger"></a>

### Details van command-logger

Logt elke slashopdracht als een JSON-regel (tijdstempel, actie, sessiesleutel, afzender-ID, bron) naar `~/.openclaw/logs/commands.log`.

<a id="compaction-notifier"></a>

### Details van compaction-notifier

Stuurt korte statusberichten naar het huidige gesprek wanneer OpenClaw begint en klaar is met het compact maken van het sessietranscript. Dit maakt lange beurten minder verwarrend in chatinterfaces, omdat de gebruiker kan zien dat de assistent de context samenvat en na de compactie doorgaat.

<a id="boot-md"></a>

### Details van boot-md

Voert `BOOT.md` uit wanneer de Gateway wordt opgestart voor elk geconfigureerd agentbereik, als het bestand bestaat in de opgeloste werkruimte van die agent.

## Pluginhooks

Plugins kunnen getypeerde hooks registreren via de Plugin SDK voor diepere integratie:
toolaanroepen onderscheppen, prompts wijzigen, de berichtenstroom beheren en meer.
Gebruik pluginhooks wanneer je `before_tool_call`, `before_agent_reply`,
`before_install` of andere levenscyclushooks binnen het proces nodig hebt.

Door plugins beheerde interne hooks zijn anders: ze nemen deel aan het grove
opdracht-/levenscyclusgebeurtenissensysteem van deze pagina en verschijnen in `openclaw hooks list` als
`plugin:<id>`. Gebruik deze voor neveneffecten en compatibiliteit met hookpakketten, niet
voor geordende middleware of beleidscontroles.

Zie [Pluginhooks](/nl/plugins/hooks) voor de volledige referentie voor pluginhooks.

## Configuratie

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

Omgevingswaarden per hook voldoen aan de geschiktheidscontroles van een hook voor `requires.env` (naast de procesomgeving), en handlers kunnen ze uit hun hookconfiguratie-item lezen:

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": { "MY_CUSTOM_VAR": "value" }
        }
      }
    }
  }
}
```

Extra hookmappen:

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

<Note>
De verouderde configuratie-indeling met de array `hooks.internal.handlers` wordt nog steeds ondersteund voor achterwaartse compatibiliteit, maar nieuwe hooks moeten het op detectie gebaseerde systeem gebruiken.
</Note>

## CLI-referentie

```bash
# Alle hooks weergeven (voeg --eligible, --verbose of --json toe)
openclaw hooks list

# Gedetailleerde informatie over een hook weergeven
openclaw hooks info <hook-name>

# Geschiktheidsoverzicht weergeven
openclaw hooks check

# Inschakelen/uitschakelen
openclaw hooks enable <hook-name>
openclaw hooks disable <hook-name>
```

## Aanbevolen werkwijzen

- **Houd handlers snel.** Hooks worden uitgevoerd tijdens de verwerking van opdrachten. Start zwaar werk zonder erop te wachten met `void processInBackground(event)`.
- **Handel fouten correct af.** Plaats risicovolle bewerkingen in try/catch; werp geen fouten op, zodat andere handlers kunnen worden uitgevoerd.
- **Filter gebeurtenissen vroegtijdig.** Keer onmiddellijk terug als het gebeurtenistype/de actie niet relevant is.
- **Gebruik specifieke gebeurtenissleutels.** Geef de voorkeur aan `"events": ["command:new"]` boven `"events": ["command"]` om overhead te verminderen.

## Probleemoplossing

### Hook niet gedetecteerd

```bash
# Mapstructuur verifiëren
ls -la ~/.openclaw/hooks/my-hook/
# Moet het volgende tonen: HOOK.md, handler.ts

# Alle gedetecteerde hooks weergeven
openclaw hooks list
```

### Hook niet geschikt

```bash
openclaw hooks info my-hook
```

Controleer op ontbrekende binaire bestanden (PATH), omgevingsvariabelen, configuratiewaarden of compatibiliteit met het besturingssysteem.

### Hook wordt niet uitgevoerd

1. Controleer of de hook is ingeschakeld: `openclaw hooks list`
2. Start je Gateway-proces opnieuw zodat de hooks opnieuw worden geladen.
3. Controleer de Gateway-logboeken: `openclaw logs --follow | grep -i hook`

## Gerelateerd

- [CLI-referentie: hooks](/nl/cli/hooks)
- [Webhooks](/nl/automation/cron-jobs#webhooks)
- [Pluginhooks](/nl/plugins/hooks) — levenscyclushooks voor plugins binnen het proces
- [Configuratie](/nl/gateway/configuration-reference#hooks)
