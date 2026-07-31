---
read_when:
    - Je moet de onbewerkte modeluitvoer controleren op het uitlekken van redeneringen
    - Je wilt de Gateway in watch-modus uitvoeren terwijl je iteratief werkt
    - Je hebt een herhaalbare workflow voor foutopsporing nodig
summary: 'Foutopsporingstools: watch-modus, onbewerkte modelstreams en het traceren van gelekte redeneringen'
title: Foutopsporing
x-i18n:
    generated_at: "2026-07-27T05:15:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45a1196c03e4deede3ce47553e1b2b3e1903ee04fe6855d929e0c32bf4e5e686
    source_path: help/debugging.md
    workflow: 16
---

Hulpmiddelen voor foutopsporing bij streaminguitvoer, Gateway-iteratie en opstartprofilering.

## Foutopsporingsoverschrijvingen tijdens runtime

`/debug` stelt configuratieoverschrijvingen **alleen voor de runtime** in (in het geheugen, niet op schijf). Standaard uitgeschakeld; schakel dit in met `commands.debug: true`.

```text
/debug show
/debug set channels.whatsapp.responsePrefix="[openclaw]"
/debug unset channels.whatsapp.responsePrefix
/debug reset
```

`/debug reset` wist alle overschrijvingen en keert terug naar de configuratie op schijf.

## Uitvoer van sessietraces

`/trace` toont trace-/foutopsporingsregels die eigendom zijn van de Plugin voor één sessie, zonder de volledig uitgebreide modus in te schakelen. Gebruik dit voor Plugin-diagnostiek, zoals foutopsporingsoverzichten van Active Memory; gebruik `/verbose` voor normale status-/tooluitvoer.

```text
/trace
/trace on
/trace off
```

## Levenscyclustrace van Plugins

Stel `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` in voor een uitsplitsing per fase van Plugin-metadata, detectie, register, runtimespiegel, configuratiewijziging en vernieuwingswerk. Schrijft naar stderr, zodat JSON-opdrachtuitvoer parseerbaar blijft.
Mislukte Plugin-ladingen bevatten hun stacktrace zolang deze trace is ingeschakeld.

```bash
OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1 openclaw plugins install tokenjuice --force
```

```text
[plugins:lifecycle] phase="config read" ms=6.83 status=ok command="install"
[plugins:lifecycle] phase="slot selection" ms=94.31 status=ok command="install" pluginId="tokenjuice"
[plugins:lifecycle] phase="registry refresh" ms=51.56 status=ok command="install" reason="source-changed"
```

Gebruik dit voordat je een CPU-profiler inzet. Meet vanuit een broncheckout de gebouwde runtime met `node dist/entry.js ...` na `pnpm build`; `pnpm openclaw ...` meet ook de overhead van de bronrunner.

Gebruik voor synchrone timing van het laden van modules het gedeelde diagnostische oppervlak in plaats van een afzonderlijke omgevingsschakelaar die alleen voor Plugins geldt:

```bash
OPENCLAW_DIAGNOSTICS=plugin.load-profile openclaw plugins list
```

## Profilering van CLI-opstart en opdrachten

Ingecheckte opstartbenchmarks:

```bash
pnpm test:startup:bench:smoke
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --runs 3
pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu
```

Stel voor eenmalige profilering via de normale bronrunner `OPENCLAW_RUN_NODE_CPU_PROF_DIR` in:

```bash
OPENCLAW_RUN_NODE_CPU_PROF_DIR=.artifacts/cli-cpu pnpm openclaw status
```

De bronrunner voegt CPU-profielvlaggen voor Node toe en schrijft een `.cpuprofile` voor de opdracht. Gebruik dit voordat je tijdelijke instrumentatie aan opdrachtcode toevoegt.

Voeg voor opstartvertragingen die op synchroon bestandssysteem- of moduleladerwerk lijken, via de bronrunner de tracevlag voor synchrone I/O van Node toe:

```bash
OPENCLAW_TRACE_SYNC_IO=1 pnpm openclaw gateway --force
```

`pnpm gateway:watch` laat deze vlag standaard uitgeschakeld voor het bewaakte onderliggende Gateway-proces; stel `OPENCLAW_TRACE_SYNC_IO=1` in als je ook in de bewakingsmodus trace-uitvoer voor synchrone I/O wilt.

## Bewakingsmodus van de Gateway

```bash
pnpm gateway:watch
```

Standaard start of herstart dit een tmux-sessie met de naam `openclaw-gateway-watch-<profile>` (bijvoorbeeld `openclaw-gateway-watch-main`), waarbij alleen een poortsuffix zoals `openclaw-gateway-watch-dev-19001` wordt toegevoegd wanneer `OPENCLAW_GATEWAY_PORT` afwijkt van de standaardpoort `18789`. Vanuit interactieve terminals wordt automatisch gekoppeld; niet-interactieve shells, CI en uitvoeraanroepen van agents blijven losgekoppeld en tonen in plaats daarvan instructies om te koppelen:

```bash
tmux attach -t openclaw-gateway-watch-main
# Recente uitvoer lezen zonder te koppelen
tmux capture-pane -ep -t openclaw-gateway-watch-main -S -200
```

Het paneel gebruikt tmux `remain-on-exit`, zodat opstartfouten beschikbaar blijven om te koppelen of vast te leggen, in plaats van de sessie te verwijderen. Door `pnpm gateway:watch` opnieuw uit te voeren, wordt dat paneel opnieuw gestart.

Het tmux-paneel voert de onbewerkte watcher uit:

```bash
node scripts/watch-node.mjs gateway --force
```

Voordat de geconfigureerde/standaardpoort wordt bewaakt, stopt de tmux-wrapper de geïnstalleerde Gateway-service van het actieve profiel. Hierdoor wordt de poort aan de bronwatcher overgedragen zonder dat launchd, systemd of Scheduled Task de service opnieuw start en vervangt. De service blijft geïnstalleerd; herstel deze na de bewakingssessie met:

```bash
pnpm openclaw gateway start
```

Wanneer een expliciete `--port` of `OPENCLAW_GATEWAY_PORT` afwijkt van de effectieve poort van de geïnstalleerde service, laat de wrapper de service actief zodat beide Gateways naast elkaar kunnen draaien.

Voorgrondmodus zonder tmux:

```bash
pnpm gateway:watch:raw
# of
OPENCLAW_GATEWAY_WATCH_TMUX=0 pnpm gateway:watch
```

De onbewerkte modus beheert de geïnstalleerde service niet. Voer eerst `pnpm openclaw gateway stop` uit wanneer deze dezelfde poort gebruikt.

Behoud tmux-beheer, maar schakel automatisch koppelen uit:

```bash
OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch
```

Profileer de CPU-tijd van de bewaakte Gateway bij het opsporen van knelpunten tijdens het opstarten of de runtime:

```bash
pnpm gateway:watch --benchmark
```

De bewakingswrapper verwerkt `--benchmark` voordat de Gateway wordt aangeroepen en schrijft bij elke beëindiging van een onderliggend Gateway-proces één V8-`.cpuprofile` onder `.artifacts/gateway-watch-profiles/`. Stop of herstart de bewaakte Gateway om het huidige profiel weg te schrijven en open het daarna met Chrome DevTools of Speedscope:

```bash
npx speedscope .artifacts/gateway-watch-profiles/*.cpuprofile
```

- `--benchmark-dir <path>`: schrijf profielen ergens anders.
- `--benchmark-no-force`: sla de standaardopschoning van poort `--force` over en stop onmiddellijk met een fout als de Gateway-poort al in gebruik is.

De benchmarkmodus onderdrukt standaard overvloedige trace-uitvoer voor synchrone I/O. Stel `OPENCLAW_TRACE_SYNC_IO=1` samen met `--benchmark` in om zowel CPU-profielen als stacktraces voor synchrone I/O te verkrijgen; in de benchmarkmodus worden die traceblokken naar `gateway-watch-output.log` onder de benchmarkmap geschreven (en uit het terminalpaneel gefilterd), terwijl normale Gateway-logboeken zichtbaar blijven.

De tmux-wrapper geeft algemene niet-geheime runtimeselectors door aan het paneel, waaronder `OPENCLAW_PROFILE`, `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`, `OPENCLAW_GATEWAY_PORT` en `OPENCLAW_SKIP_CHANNELS`. Plaats providerreferenties in je normale profiel/configuratie of gebruik de onbewerkte voorgrondmodus voor eenmalige tijdelijke geheimen.

Als de bewaakte Gateway tijdens het opstarten wordt afgesloten, voert de watcher `openclaw doctor --fix --non-interactive` eenmaal uit en herstart deze het onderliggende Gateway-proces. Stel `OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` in om de oorspronkelijke opstartfout te zien zonder de herstelstap die alleen voor ontwikkeling is bedoeld.

Het beheerde tmux-paneel gebruikt standaard gekleurde Gateway-logboeken; stel `FORCE_COLOR=0` in bij het starten van `pnpm gateway:watch` om ANSI-uitvoer uit te schakelen.

De watcher herstart bij wijzigingen in bouwrelevante bestanden onder `src/`, bronbestanden van extensies, extensiemetadata in `package.json` en `openclaw.plugin.json`, `tsconfig.json`, `package.json` en `tsdown.config.ts`. Wijzigingen in extensiemetadata herstarten de Gateway zonder een herbouw af te dwingen; bij bron- en configuratiewijzigingen wordt nog steeds eerst `dist` herbouwd.

Voeg CLI-vlaggen voor de Gateway toe na `gateway:watch`; deze worden bij elke herstart doorgegeven. Als dezelfde bewakingsopdracht opnieuw wordt uitgevoerd, wordt het benoemde tmux-paneel opnieuw gestart; de onbewerkte watcher gebruikt een vergrendeling voor één watcher, zodat dubbele bovenliggende watcherprocessen worden vervangen in plaats van zich op te stapelen.

## Ontwikkelprofiel + ontwikkel-Gateway (--dev)

Twee **afzonderlijke** `--dev`-vlaggen:

- **Globale `--dev` (profiel):** isoleert de status onder `~/.openclaw-dev` en stelt de standaardpoort van de Gateway in op `19001` (afgeleide poorten verschuiven mee).
- **`gateway --dev`:** instrueert de Gateway om automatisch een standaardconfiguratie en werkruimte te maken wanneer die ontbreken (en bootstrap over te slaan).

Aanbevolen werkwijze (ontwikkelprofiel + ontwikkelbootstrap):

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

Voer de CLI zonder globale installatie uit via `pnpm openclaw ...`.

Wat dit doet:

1. **Profielisolatie** (globale `--dev`)
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001` (browser-/canvaspoorten verschuiven overeenkomstig)

2. **Ontwikkelbootstrap** (`gateway --dev`)
   - Schrijft een minimale configuratie als die ontbreekt (`gateway.mode=local`, koppeling aan loopback).
   - Stelt `agents.defaults.workspace` in op de ontwikkelwerkruimte en `agents.defaults.skipBootstrap=true`.
   - Maakt de werkruimtebestanden aan als ze ontbreken: `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`.
   - Standaardidentiteit: **C3-PO** (protocol-droid).
   - `pnpm gateway:dev` stelt ook `OPENCLAW_SKIP_CHANNELS=1` in om kanaalproviders over te slaan.

Ontwikkel-Gateways negeren standaard omgevingsvariabelen die kanalen activeren, zodat referenties die uit je shell worden overgenomen de ontwikkelinstantie niet verbinden met echte kanaalservices. Expliciete configuratie via `channels.<id>` blijft werken. Geef `--dev-ambient-channels` samen met `--dev` door om de automatische kanaalconfiguratie vanuit de omgeving voor die uitvoering te herstellen.

Herstelwerkwijze (nieuwe start):

```bash
pnpm gateway:dev:reset
```

<Note>
`--dev` is een **globale** profielvlag en wordt door sommige runners verwerkt en verwijderd. Gebruik de vorm met de omgevingsvariabele als je deze expliciet moet opgeven:

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

</Note>

`--reset` wist de configuratie, referenties, sessies en de ontwikkelwerkruimte (verplaatst naar de prullenbak, niet verwijderd) en maakt vervolgens de standaardontwikkelomgeving opnieuw aan.

<Tip>
Als er al een niet-ontwikkel-Gateway actief is (launchd of systemd), stop deze dan eerst:

```bash
openclaw gateway stop
```

</Tip>

## Logboekregistratie van onbewerkte streams

OpenClaw kan de **onbewerkte assistentstream** registreren voordat filtering of opmaak plaatsvindt. Dit is de beste manier om te zien of redeneringen binnenkomen als delta's met platte tekst (of als afzonderlijke denkblokken).

Schakel dit in via de CLI:

```bash
pnpm gateway:watch --raw-stream
```

Optionele padoverschrijving:

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

Overeenkomstige omgevingsvariabelen:

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

Standaardbestand: `~/.openclaw/logs/raw-stream.jsonl`

## Veiligheidsopmerkingen

- Logboeken van onbewerkte streams kunnen volledige prompts, tooluitvoer en gebruikersgegevens bevatten.
- Bewaar logboeken lokaal en verwijder ze na het foutopsporen.
- Verwijder eerst geheimen en persoonsgegevens als je logboeken deelt.

## Foutopsporing in VSCode

Bronkaarten zijn vereist omdat de build gegenereerde bestandsnamen hasht. De meegeleverde `launch.json` is gericht op de Gateway-service:

1. **Rebuild and Debug Gateway** - verwijdert `/dist` en bouwt opnieuw met foutopsporing ingeschakeld voordat de Gateway wordt gestart.
2. **Debug Gateway** - spoort fouten op in een bestaande build zonder `/dist` te wijzigen.

### Instellen

1. Open **Run and Debug** (Activity Bar of `Ctrl`+`Shift`+`D`).
2. Selecteer **Rebuild and Debug Gateway** en druk op **Start Debugging**.

Om de bouw-/foutopsporingscyclus in plaats daarvan handmatig te beheren:

1. Schakel bronkaarten in een terminal in:
   - **Linux/macOS**: `export OUTPUT_SOURCE_MAPS=1`
   - **Windows (PowerShell)**: `$env:OUTPUT_SOURCE_MAPS="1"`
   - **Windows (CMD)**: `set OUTPUT_SOURCE_MAPS=1`
2. Bouw opnieuw: `pnpm clean:dist && pnpm build`
3. Selecteer **Debug Gateway** en druk op **Start Debugging**.

Stel onderbrekingspunten in `src/` TypeScript-bestanden in; de debugger koppelt deze via bronkaarten aan gecompileerde JavaScript.

### Opmerkingen

- **Rebuild and Debug Gateway** verwijdert `/dist` en voert bij elke start een volledige `pnpm build` met bronkaarten uit.
- **Debug Gateway** kan starten en stoppen zonder `/dist` te beïnvloeden, maar je beheert de bouwcyclus in een afzonderlijke terminal.
- Bewerk `launch.json` `args` om fouten in andere CLI-subopdrachten op te sporen.
- Als je de gebouwde CLI voor andere taken wilt gebruiken (bijvoorbeeld `dashboard --no-open` als je foutopsporingssessie een nieuw authenticatietoken genereert), voer je deze uit vanuit een andere terminal: `node ./openclaw.mjs` of een alias zoals `alias openclaw-build="node $(pwd)/openclaw.mjs"`.

## Gerelateerd

- [Problemen oplossen](/nl/help/troubleshooting)
- [Veelgestelde vragen](/nl/help/faq)
