---
read_when:
    - Loguitvoer of -indelingen wijzigen
    - CLI- of Gateway-uitvoer debuggen
summary: Logboekoppervlakken, bestandslogboeken, WS-logboekstijlen en consoleopmaak
title: Gateway-logboekregistratie
x-i18n:
    generated_at: "2026-07-27T05:04:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0b11a68611032c29c31091b2411982487e7f5df3ecf4f1e3b586e7d21e543d3
    source_path: gateway/logging.md
    workflow: 16
---

# Logboekregistratie

Zie [/logging](/nl/logging) voor een gebruikersgericht overzicht (CLI + Control UI + configuratie).

OpenClaw heeft twee logboekoppervlakken:

- **Console-uitvoer** - wat je in de terminal / Debug UI ziet.
- **Bestandslogboeken** - JSON-regels die door de Gateway-logger worden geschreven.

Bij het opstarten registreert de Gateway het herleide standaardmodel van de agent plus de modusstandaarden die van invloed zijn op nieuwe sessies:

```text
agentmodel: openai/gpt-5.6-sol (denken=gemiddeld, snel=aan)
```

`thinking` is afkomstig van de standaardagent, modelparameters of de algemene standaardwaarde voor agents; wanneer dit niet is ingesteld, wordt `medium` weergegeven. `fast` is afkomstig van de standaardagent of de `fastMode`-parameters van het model.

## Bestandslogger

- Standaard staan roulerende logboekbestanden onder `/tmp/openclaw/` (één bestand per dag), gedateerd volgens de lokale tijdzone van de Gateway-host. Het standaardprofiel gebruikt `openclaw-YYYY-MM-DD.log`; benoemde profielen gebruiken `openclaw-<profile>-YYYY-MM-DD.log` (bijvoorbeeld `openclaw-dev-YYYY-MM-DD.log`). Als die map onveilig of niet beschrijfbaar is (verkeerde eigenaar, schrijfbaar voor iedereen, een symbolische koppeling), valt OpenClaw terug op een gebruikersgebonden pad onder `os.tmpdir()/openclaw-<uid>`; op Windows gebruikt het altijd die terugvaloptie via de tijdelijke map van het besturingssysteem.
- Actieve logboekbestanden rouleren bij `logging.maxFileBytes` (standaard: 100 MB), waarbij maximaal vijf genummerde archieven (`.1` tot en met `.5`) worden bewaard en verder wordt geschreven naar een nieuw actief bestand.
- Configureer het pad en niveau van het logboekbestand via `~/.openclaw/openclaw.json`: `logging.file`, `logging.level`.
- De bestandsindeling bestaat uit één JSON-object per regel.

Codepaden voor gesprekken, realtime spraak en beheerde ruimten gebruiken de gedeelde bestandslogger voor begrensde levenscyclusregistraties die bedoeld zijn voor operationele foutopsporing en de export van OTLP-logboekregistraties. Transcripttekst, audioladingen, beurt-id's, oproep-id's en item-id's van providers worden nooit naar de logboekregistratie gekopieerd.

Het tabblad Logs van de Control UI volgt dit bestand via de Gateway (`logs.tail`). De CLI doet hetzelfde:

```bash
openclaw logs --follow
```

### Uitgebreide uitvoer versus logboekniveaus

- **Bestandslogboeken** worden uitsluitend beheerd door `logging.level`.
- `--verbose` beïnvloedt alleen de **uitgebreidheid van de console-uitvoer** (en de stijl van WS-logboeken) - het verhoogt **niet** het niveau van het bestandslogboek.
- Stel `logging.level` in op `debug` of `trace` om details die alleen bij uitgebreide uitvoer verschijnen in bestandslogboeken vast te leggen.
- Trace-logboekregistratie bevat ook diagnostische tijdsduuroverzichten voor geselecteerde intensief gebruikte codepaden, zoals het voorbereiden van de factory voor Plugin-tools. Zie [/tools/plugin#slow-plugin-tool-setup](/nl/tools/plugin#slow-plugin-tool-setup).

## Console vastleggen

De CLI legt `console.log/info/warn/error/debug/trace` vast, schrijft ze naar bestandslogboeken en drukt ze nog steeds af naar stdout/stderr.

Stem de uitgebreidheid van de console-uitvoer afzonderlijk af:

- `logging.consoleLevel` (standaard `info`)
- `logging.consoleStyle` (`pretty` | `compact` | `json`; standaard `pretty` op een TTY, anders `compact`)

## Redactie

OpenClaw maskeert gevoelige tokens voordat logboek- of transcriptuitvoer het proces verlaat. Dit redactiebeleid is van toepassing op tekstuitvoerdoelen voor de console, bestandslogboeken, OTLP-logboekregistraties en sessietranscripten, zodat overeenkomende geheime waarden worden gemaskeerd voordat JSONL-regels of berichten naar schijf worden geschreven.

- Redactie van gevoelige waarden is altijd ingeschakeld.
- `logging.redactPatterns`: reeks regex-tekenreeksen (overschrijft standaardwaarden)
  - Gebruik onbewerkte regex-tekenreeksen (automatisch `gi`) of `/pattern/flags` voor aangepaste vlaggen.
  - Overeenkomsten worden gemaskeerd met behoud van de eerste 6 + laatste 4 tekens (waarden van >= 18 tekens); kortere waarden worden `***`.
  - De standaardwaarden omvatten veelvoorkomende sleuteltoewijzingen, CLI-vlaggen, JSON-velden, bearer-headers, PEM-blokken, populaire voorvoegsels van tokens van leveranciers en veldnamen voor betalingsreferenties (kaartnummer, CVC/CVV, gedeeld betalingstoken, betalingsreferentie).

Veiligheidsgrenzen zoals toolaanroepgebeurtenissen in de Control UI, uitvoer van `sessions_history`, diagnostische exports, providerfouten, de weergave van uitvoeringsgoedkeuringen en Gateway-WebSocket-logboeken passen altijd redactie toe. `logging.redactPatterns` voegt implementatiespecifieke patronen toe.

## Gateway-WebSocket-logboeken

De Gateway drukt WebSocket-protocollogboeken in twee modi af:

- **Normale modus (zonder `--verbose`)**: alleen ‘interessante’ RPC-resultaten worden afgedrukt - fouten (`ok=false`), trage aanroepen (standaarddrempel: `>= 50ms`) en parseerfouten.
- **Uitgebreide modus (`--verbose`)**: drukt al het WS-verzoek-/antwoordverkeer af.

### WS-logboekstijl

`openclaw gateway` ondersteunt een stijlschakelaar per Gateway:

- `--ws-log auto` (standaard): de normale modus is geoptimaliseerd; de uitgebreide modus gebruikt compacte uitvoer.
- `--ws-log compact`: compacte uitvoer (gekoppeld verzoek/antwoord) bij uitgebreide uitvoer.
- `--ws-log full`: volledige uitvoer per frame bij uitgebreide uitvoer.
- `--compact`: alias voor `--ws-log compact`.

```bash
# geoptimaliseerd (alleen fouten/traag)
openclaw gateway

# al het WS-verkeer tonen (gekoppeld)
openclaw gateway --verbose --ws-log compact

# al het WS-verkeer tonen (volledige metagegevens)
openclaw gateway --verbose --ws-log full
```

## Consoleopmaak (logboekregistratie per subsysteem)

De consoleformatter is **TTY-bewust** en drukt consistente regels met voorvoegsels af. Subsysteemloggers houden de uitvoer gegroepeerd en overzichtelijk:

- **Subsysteemvoorvoegsels** op elke regel (bijvoorbeeld `[gateway]`, `[canvas]`, `[tailscale]`).
- **Subsysteemkleuren** (stabiel per subsysteem, gehasht op basis van de naam) plus niveaukleuren.
- **Kleur wanneer de uitvoer een TTY is** of de omgeving op een terminal met uitgebreide mogelijkheden lijkt (`TERM`/`COLORTERM`/`TERM_PROGRAM`); respecteert `NO_COLOR` en `FORCE_COLOR`.
- **Verkorte subsysteemvoorvoegsels**: verwijdert een vooraanstaand segment `gateway/`, `channels/` of `providers/` en behoudt daarna maximaal de laatste 2 resterende segmenten (bijvoorbeeld `channels/turn/kernel` wordt weergegeven als `turn/kernel`). Bekende kanaalsubsystemen (`telegram`, `whatsapp`, `slack`, enzovoort) worden altijd ingekort tot alleen de kanaalnaam.
- **Subloggers per subsysteem** (automatisch voorvoegsel + gestructureerd veld `{ subsystem }`).
- **`logRaw()`** voor QR-/UX-uitvoer (geen voorvoegsel, geen opmaak).
- **Consolestijlen**: `pretty` | `compact` | `json`.
- **Consolelogboekniveau** staat los van het niveau van het bestandslogboek (het bestand behoudt alle details wanneer `logging.level` `debug`/`trace` is).
- **WhatsApp-berichtinhoud** wordt geregistreerd op `debug` (gebruik `--verbose` om deze te zien).

Hierdoor blijven bestandslogboeken stabiel en blijft interactieve uitvoer overzichtelijk.

## Gerelateerd

- [Logboekregistratie](/nl/logging)
- [OpenTelemetry-export](/nl/gateway/opentelemetry)
- [Diagnostische export](/nl/gateway/diagnostics)
