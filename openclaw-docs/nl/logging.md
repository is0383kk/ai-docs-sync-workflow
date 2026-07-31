---
read_when:
    - Je hebt een beginnersvriendelijk overzicht van OpenClaw-logboekregistratie nodig
    - Je wilt logniveaus, indelingen of redactie configureren
    - Je probeert een probleem op te lossen en moet snel logboeken vinden
summary: Bestandslogboeken, console-uitvoer, CLI-logweergave en het tabblad Logs in de Control UI
title: Logboekregistratie
x-i18n:
    generated_at: "2026-07-27T05:37:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c9303c4bc9c0797ca9c5775a281dce95229661b61d710425b2f7bec182b2e75
    source_path: logging.md
    workflow: 16
---

OpenClaw heeft twee belangrijke logoppervlakken:

- **Bestandslogs** (JSON-regels) die door de Gateway worden geschreven.
- **Console-uitvoer** in de terminal waarin de Gateway draait.

Het tabblad **Logboeken** van de Control UI volgt het logbestand van de Gateway. Op deze pagina wordt uitgelegd waar
logs worden opgeslagen, hoe je ze leest en hoe je logniveaus en -indelingen configureert.

## Waar logs worden opgeslagen

Standaard schrijft de Gateway per dag een roterend logbestand. Het standaardprofiel
behoudt het historische pad:

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

Benoemde profielen gebruiken een bestandsnaam met profielaanduiding in dezelfde map:

`/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`

Het profielsegment van de bestandsnaam bestaat uit kleine letters en is beperkt tot letters, cijfers en
streepjes. Eenvoudige namen in kleine letters blijven leesbaar, zodat de afkorting `--dev`
naar `openclaw-dev-YYYY-MM-DD.log` schrijft. Hoofdletters, underscores en letterlijke streepjes gebruiken een
omkeerbare escape met streepjes, zodat verschillende profielnamen nooit hetzelfde logbestand delen.
Te lange waarden die rechtstreeks via de omgeving worden ingesteld, krijgen een begrensd hashachtervoegsel
om binnen de limieten voor bestandsnamen van het bestandssysteem te blijven. Een expliciete `logging.file` overschrijft
deze standaardwaarden.

De datum gebruikt de lokale tijdzone van de Gateway-host. Wanneer `/tmp/openclaw` onveilig
of niet beschikbaar is (en altijd op Windows), gebruikt OpenClaw in plaats daarvan een gebruikersspecifieke
map `openclaw-<uid>` onder de tijdelijke map van het besturingssysteem. Gedateerde logbestanden worden
na 24 uur verwijderd.

Elk bestand roteert wanneer de volgende schrijfbewerking `logging.maxFileBytes` zou overschrijden
(standaard: 100 MB). OpenClaw bewaart maximaal vijf genummerde archieven naast het
actieve bestand, zoals `openclaw-YYYY-MM-DD.1.log` of
`openclaw-dev-YYYY-MM-DD.1.log`, en blijft naar een nieuw actief logbestand schrijven
in plaats van diagnostische gegevens te onderdrukken.

Je kunt het pad in `~/.openclaw/openclaw.json` overschrijven:

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## Logs lezen

### CLI: live volgen (aanbevolen)

Volg het Gateway-logbestand via RPC:

```bash
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
```

De profielselector op hoofdniveau vindt hetzelfde profielspecifieke bestand dat door de
Gateway wordt gebruikt, inclusief terugvallezingen door de CLI wanneer lokale RPC niet beschikbaar is.

Opties:

| Vlag                | Standaard  | Gedrag                                                                              |
| ------------------- | -------- | ------------------------------------------------------------------------------------- |
| `--follow`          | uit      | Blijven volgen; maakt na een verbroken verbinding met oplopende wachttijd opnieuw verbinding                                   |
| `--limit <n>`       | `200`    | Maximumaantal regels per ophaalbewerking                                                                   |
| `--max-bytes <n>`   | `250000` | Maximumaantal bytes dat per ophaalbewerking wordt gelezen                                                           |
| `--interval <ms>`   | `1000`   | Pollinterval tijdens het volgen                                                         |
| `--json`            | uit      | JSON met één gebeurtenis per regel                                              |
| `--plain`           | uit      | Platte tekst afdwingen in TTY-sessies                                                      |
| `--no-color`        | —        | ANSI-kleuren uitschakelen                                                                   |
| `--utc`             | uit      | Tijdstempels in UTC weergeven (lokale tijd is standaard)                                      |
| `--local-time`      | uit      | Geaccepteerde compatibele spelling voor de standaard lokale tijd; heeft verder geen effect       |
| `--url` / `--token` | —        | Standaardvlaggen voor Gateway-RPC                                                            |
| `--timeout <ms>`    | `30000`  | Time-out voor Gateway-RPC                                                                   |
| `--expect-final`    | uit      | Vlag voor wachten op het definitieve RPC-antwoord via een agent (hier geaccepteerd via de gedeelde clientlaag) |

Uitvoermodi:

- **TTY-sessies**: fraai opgemaakte, gekleurde, gestructureerde logregels.
- **Niet-TTY-sessies**: platte tekst.

Wanneer je een expliciete `--url` opgeeft, past de CLI configuratie of
omgevingsreferenties niet automatisch toe; voeg zelf `--token` toe, anders mislukt de aanroep met
`gateway url override requires explicit credentials`.

In JSON-modus geeft de CLI met `type` gelabelde objecten uit:

- `meta`: streammetadata (bestand, bron, brontype, service, cursor, grootte)
- `log`: geparseerd logitem
- `notice`: aanwijzingen voor afkapping/rotatie
- `raw`: niet-geparseerde logregel
- `error`: verbindingsfouten met de Gateway (naar stderr geschreven)

Als de impliciete lokale loopback-Gateway om koppeling vraagt, tijdens het verbinden
sluit of een time-out bereikt voordat `logs.tail` antwoordt, valt `openclaw logs` automatisch terug op het
geconfigureerde Gateway-logbestand. Expliciete `--url`-doelen gebruiken
deze terugval niet. `openclaw logs --follow` is strenger: op Linux gebruikt het indien beschikbaar
het actieve Gateway-journal van de gebruiker in systemd op basis van PID, en anders probeert het
de live Gateway met oplopende wachttijd opnieuw in plaats van een mogelijk verouderd bestand ernaast
te volgen.

Als de Gateway onbereikbaar is, toont de CLI een korte aanwijzing om dit uit te voeren:

```bash
openclaw doctor
```

### Control UI (web)

Het tabblad **Logboeken** van de Control UI volgt hetzelfde bestand met `logs.tail`.
Zie [Control UI](/nl/web/control-ui) voor informatie over het openen ervan.

### Logs voor alleen kanalen

Gebruik het volgende om kanaalactiviteit (WhatsApp/Telegram/enz.) te filteren:

```bash
openclaw channels logs --channel whatsapp
```

`--channel` is standaard `all`; `--lines <n>` (standaard 200) en `--json` zijn ook
beschikbaar.

## Logindelingen

### Bestandslogs (JSONL)

Elke regel in het logbestand is een JSON-object. De CLI en Control UI parseren deze
items om gestructureerde uitvoer weer te geven (tijd, niveau, subsysteem, bericht).

JSONL-records in bestandslogs bevatten indien beschikbaar ook machinaal filterbare velden op het hoogste niveau:

- `hostname`: hostnaam van de Gateway.
- `message`: afgevlakte tekst van het logbericht voor zoeken in volledige tekst.
- `agent_id`: actieve agent-id wanneer de logaanroep agentcontext bevat.
- `session_id`: actieve sessie-id/-sleutel wanneer de logaanroep sessiecontext bevat.
- `channel`: actief kanaal wanneer de logaanroep kanaalcontext bevat.

OpenClaw behoudt de oorspronkelijke gestructureerde logargumenten naast deze velden,
zodat bestaande parsers die genummerde tslog-argumentsleutels lezen, blijven werken.

Activiteit voor Talk, realtime spraak en beheerde ruimtes genereert begrensde levenscycluslogrecords
via dezelfde pijplijn voor bestandslogs. Deze records bevatten gebeurtenistype,
modus, transport, provider en indien beschikbaar metingen van grootte/timing, maar laten
transcripttekst, audiopayloads, beurt-id's, oproep-id's en provideritem-id's weg.

### Console-uitvoer

Consolelogs zijn **TTY-bewust** en opgemaakt voor leesbaarheid:

- Voorvoegsels van subsystemen (bijv. `gateway/channels/whatsapp`)
- Kleuren voor niveaus (info/warn/error)
- Optionele compacte modus of JSON-modus

De consoleopmaak wordt geregeld door `logging.consoleStyle`.

### Gateway-WebSocket-logs

`openclaw gateway` heeft ook WebSocket-protocollogging voor RPC-verkeer:

- normale modus: alleen interessante resultaten (fouten, parseerfouten, trage aanroepen)
- `--verbose`: al het aanvraag-/antwoordverkeer
- `--ws-log auto|compact|full`: kies de uitgebreide weergavestijl
- `--compact`: alias voor `--ws-log compact`

Voorbeelden:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## Logging configureren

Alle loggingconfiguratie staat onder `logging` in `~/.openclaw/openclaw.json`.

```json
{
  "logging": {
    "level": "info",
    "file": "/path/to/openclaw.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### Logniveaus

Niveaus: `silent`, `fatal`, `error`, `warn`, `info`, `debug`, `trace`.

- `logging.level`: niveau voor **bestandslogs** (JSONL) (standaard: `info`).
- `logging.consoleLevel`: uitvoerigheidsniveau van de **console**.

Je kunt beide overschrijven via de omgevingsvariabele **`OPENCLAW_LOG_LEVEL`** (bijv. `OPENCLAW_LOG_LEVEL=debug`). De omgevingsvariabele heeft voorrang op het configuratiebestand, zodat je de uitvoerigheid voor één uitvoering kunt verhogen zonder `openclaw.json` te bewerken. Je kunt ook de algemene CLI-optie **`--log-level <level>`** meegeven (bijvoorbeeld `openclaw --log-level debug gateway run`), die voor die opdracht de omgevingsvariabele overschrijft.

`--verbose` heeft alleen invloed op console-uitvoer en de uitvoerigheid van WS-logs; het wijzigt
de niveaus van bestandslogs niet.

### Gerichte diagnostiek voor modeltransport

Gebruik bij het opsporen van fouten in provideraanroepen gerichte omgevingsvlaggen in plaats van
alle logs te verhogen naar `debug`:

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools OPENCLAW_DEBUG_SSE=events openclaw gateway
```

Beschikbare vlaggen:

- `OPENCLAW_DEBUG_MODEL_TRANSPORT=1`: registreer het begin van de aanvraag, het fetch-antwoord, SDK-
  headers, de eerste streaminggebeurtenis, de voltooiing van de stream en transportfouten op
  niveau `info`.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=summary`: neem een begrensde samenvatting van de aanvraagpayload
  op in logboeken van modelaanvragen.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=tools`: neem alle namen van modelgerichte tools op in
  de payloadsamenvatting.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`: neem een geredigeerde, in grootte beperkte JSON-
  momentopname van de payload op. Gebruik dit alleen tijdens foutopsporing; geheimen worden geredigeerd, maar prompts
  en berichttekst kunnen nog steeds aanwezig zijn.
- `OPENCLAW_DEBUG_SSE=events`: registreer de timing van de eerste gebeurtenis en de voltooiing van de stream.
- `OPENCLAW_DEBUG_SSE=peek`: registreer ook de eerste vijf geredigeerde SSE-gebeurtenispayloads,
  met een limiet per gebeurtenis.
- `OPENCLAW_DEBUG_CODE_MODE=1`: registreer diagnostiek voor het modeloppervlak in codemodus,
  ook wanneer systeemeigen providertools verborgen zijn omdat de codemodus het
  tooloppervlak beheert.

Deze vlaggen registreren via de normale logging van OpenClaw, zodat `openclaw logs --follow`
en het tabblad Logboeken van de Control UI ze tonen. Zonder de vlaggen blijft dezelfde diagnostiek
beschikbaar op niveau `debug`.

`[model-fetch]`-metadata over begin en antwoord (provider, API, model, status,
latentie en aanvraagvelden zoals methode, URL, time-out, proxy en beleid)
wordt altijd geregistreerd op niveau `info`, ongeacht
`OPENCLAW_DEBUG_MODEL_TRANSPORT`, zodat basale controle van modeltransport zichtbaar is
zonder foutopsporingsvlaggen.

### Tracecorrelatie

Bestandslogs zijn JSONL. Wanneer een logaanroep een geldige diagnostische tracecontext bevat,
schrijft OpenClaw de tracevelden als JSON-sleutels op het hoogste niveau (`traceId`, `spanId`,
`parentSpanId`, `traceFlags`), zodat externe logverwerkers de regel kunnen correleren
met OTEL-spans en propagatie van provider-`traceparent`.

Gateway-HTTP-aanvragen en Gateway-WebSocket-frames stellen een intern tracebereik voor de aanvraag
in. Logs en diagnostische gebeurtenissen die binnen dat asynchrone bereik worden gegenereerd, nemen
de aanvraagtrace over wanneer ze geen expliciete tracecontext doorgeven. Traces van agentuitvoeringen en
modelaanroepen worden onderliggende traces van de actieve aanvraagtrace, zodat lokale logs,
diagnostische momentopnamen, OTEL-spans en vertrouwde providerheaders voor `traceparent`
via `traceId` kunnen worden gekoppeld zonder onbewerkte aanvraag- of modelinhoud te registreren.

Talk-levenscycluslogrecords worden ook naar de diagnostics-otel-logexport gestuurd wanneer
OpenTelemetry-logexport is ingeschakeld, met dezelfde begrensde attributen als bestandslogs.
Configureer `diagnostics.otel.logsExporter` om OTLP, stdout-JSONL of
beide doelen te kiezen.

### Grootte en timing van modelaanroepen

Diagnostiek voor modelaanroepen registreert begrensde metingen van aanvragen/antwoorden zonder
onbewerkte prompt- of antwoordinhoud vast te leggen:

- `requestPayloadBytes`: UTF-8-bytegrootte van de uiteindelijke payload van het modelverzoek
- `responseStreamBytes`: UTF-8-bytegrootte van een gestreamd fragment van het modelantwoord
  payloads. Hoogfrequente tekst-, denk- en toolaanroep-deltagebeurtenissen tellen
  alleen de incrementele `delta`-bytes in plaats van volledige `partial`-momentopnamen.
- `timeToFirstByteMs`: verstreken tijd vóór de eerste gestreamde antwoordgebeurtenis
- `durationMs`: totale duur van de modelaanroep

Deze velden zijn beschikbaar voor diagnostische momentopnamen, Plugin-hooks voor modelaanroepen en
OTEL-spans/-metrieken voor modelaanroepen wanneer diagnostische export is ingeschakeld.

### Consolestijlen

`logging.consoleStyle`:

- `pretty`: gebruiksvriendelijk, gekleurd en met tijdstempels.
- `compact`: compactere uitvoer (het meest geschikt voor lange sessies).
- `json`: JSON per regel (voor logverwerkers).

### Redactie

OpenClaw kan gevoelige tokens redigeren voordat ze terechtkomen in console-uitvoer, bestandslogs,
OTLP-logrecords, opgeslagen sessietranscripttekst of payloads van toolgebeurtenissen
in de Control UI (argumenten bij het starten van tools, gedeeltelijke/definitieve resultaatpayloads, afgeleide
exec-uitvoer en patchsamenvattingen):

- Redactie van gevoelige waarden is altijd ingeschakeld.
- `logging.redactPatterns`: lijst met regex-tekenreeksen die de standaardset voor log-/transcriptuitvoer vervangt. Voor toolpayloads van de Control UI worden aangepaste patronen boven op de ingebouwde standaardpatronen toegepast, zodat het toevoegen van een patroon nooit de redactie verzwakt van waarden die al door de standaardpatronen worden gedetecteerd.

Bestandslogs en sessietranscripten blijven JSONL, maar overeenkomende geheime waarden worden
gemaskeerd voordat de regel of het bericht naar schijf wordt geschreven. Redactie gebeurt naar beste vermogen:
deze wordt toegepast op tekstbevattende berichtinhoud en logtekenreeksen, niet op elk
identificatieveld of binair payloadveld.

De ingebouwde standaardpatronen dekken gangbare API-aanmeldgegevens en veldnamen voor
betalingsgegevens, zoals kaartnummer, CVC/CVV, gedeeld betalingstoken en betalingsgegevens,
wanneer deze voorkomen als JSON-velden, URL-parameters, CLI-vlaggen of toewijzingen.

OpenClaw redigeert ook payloads aan de veiligheidsgrens die worden getoond aan UI-clients, ondersteuningsbundels,
diagnostische waarnemers, goedkeuringsprompts of agenttools. Aangepaste
`logging.redactPatterns` kunnen projectspecifieke patronen aan die oppervlakken toevoegen.

## Diagnostiek en OpenTelemetry

Diagnostiek bestaat uit gestructureerde, machineleesbare gebeurtenissen voor modeluitvoeringen en
telemetrie van berichtstromen (webhooks, wachtrijvorming, sessiestatus). Deze gebeurtenissen vervangen
logs **niet** — ze voeden metrieken, traces en exporters. Gebeurtenissen worden
standaard binnen het proces uitgezonden (stel `diagnostics.enabled: false` in om ze uit te schakelen);
de export ervan wordt afzonderlijk geregeld.

Twee aangrenzende oppervlakken:

- **OpenTelemetry-export** — stuur metrieken, traces en logs via OTLP/HTTP naar
  elke OpenTelemetry-compatibele collector of backend (Datadog, Grafana,
  Honeycomb, New Relic, Tempo enzovoort). De volledige configuratie, signaalcatalogus,
  namen van metrieken/spans, omgevingsvariabelen en het privacymodel staan op een speciale pagina:
  [OpenTelemetry-export](/nl/gateway/opentelemetry).
- **Diagnostiekvlaggen** — gerichte vlaggen voor debuglogs die extra logs naar
  `logging.file` sturen zonder `logging.level` te verhogen. Vlaggen zijn niet hoofdlettergevoelig
  en ondersteunen jokertekens (`telegram.*`, `*`). Configureer ze onder `diagnostics.flags`
  of via de omgevingsoverschrijving `OPENCLAW_DIAGNOSTICS=...`. Volledige handleiding:
  [Diagnostiekvlaggen](/nl/diagnostics/flags).

Zie [OpenTelemetry-export](/nl/gateway/opentelemetry) voor OTLP-export naar een collector.

## Tips voor probleemoplossing

- **Gateway niet bereikbaar?** Voer eerst `openclaw doctor` uit.
- **Logs leeg?** Controleer of de Gateway actief is en naar het bestandspad
  in `logging.file` schrijft.
- **Meer details nodig?** Stel `logging.level` in op `debug` of `trace` en probeer het opnieuw.

## Gerelateerd

- [OpenTelemetry-export](/nl/gateway/opentelemetry) — OTLP/HTTP-export, catalogus van metrieken/spans, privacymodel
- [Diagnostiekvlaggen](/nl/diagnostics/flags) — gerichte vlaggen voor debuglogs
- [Interne werking van Gateway-logboekregistratie](/nl/gateway/logging) — WS-logstijlen, voorvoegsels van subsystemen en consolevastlegging
- [Configuratiereferentie](/nl/gateway/configuration-reference#diagnostics) — volledige referentie van het veld `diagnostics.*`
