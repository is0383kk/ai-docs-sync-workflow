---
read_when:
    - De Gateway uitvoeren vanuit de CLI (ontwikkeling of servers)
    - Foutopsporing voor Gateway-authenticatie, bindmodi en connectiviteit
    - Gateways ontdekken via Bonjour (lokaal + wide-area DNS-SD)
    - Een externe procesbeheerder voor de Gateway integreren
sidebarTitle: Gateway
summary: OpenClaw Gateway-CLI (`openclaw gateway`) — gateways uitvoeren, opvragen en ontdekken
title: Gateway
x-i18n:
    generated_at: "2026-07-27T05:27:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

De Gateway is de WebSocket-server van OpenClaw (kanalen, nodes, sessies, hooks). Alle onderstaande subopdrachten vallen onder `openclaw gateway ...`.

<CardGroup cols={3}>
  <Card title="Bonjour-detectie" href="/nl/gateway/bonjour">
    Lokale mDNS- en wide-area DNS-SD-configuratie.
  </Card>
  <Card title="Overzicht van detectie" href="/nl/gateway/discovery">
    Hoe OpenClaw gateways aankondigt en vindt.
  </Card>
  <Card title="Configuratie" href="/nl/gateway/configuration">
    Configuratiesleutels op het hoogste niveau voor de Gateway.
  </Card>
</CardGroup>

## De Gateway uitvoeren

```bash
openclaw gateway
openclaw gateway run   # gelijkwaardige, expliciete vorm
```

<AccordionGroup>
  <Accordion title="Opstartgedrag">
    - Weigert te starten tenzij `gateway.mode=local` is ingesteld in `~/.openclaw/openclaw.json`. Gebruik `--allow-unconfigured` voor ad-hoc-/ontwikkeluitvoeringen; dit omzeilt de beveiliging zonder configuratie te schrijven of te repareren.
    - Wanneer bij het opstarten een herstelbare ongeldige configuratie wordt gevonden, biedt een interactieve terminal aan om `openclaw doctor --fix` uit te voeren en wordt na toestemming eenmaal opnieuw geprobeerd op te starten. Niet-interactieve uitvoeringen repareren nooit automatisch; ze tonen in plaats daarvan de opdracht. Als de gerepareerde configuratie nog steeds ongeldig is, blijft het opstarten geblokkeerd.
    - `openclaw onboard --mode local` en `openclaw setup` schrijven `gateway.mode=local`. Als het configuratiebestand bestaat maar `gateway.mode` ontbreekt, wordt dit behandeld als een beschadigde/overschreven configuratie en weigert de Gateway `local` voor je te raden — voer de onboarding opnieuw uit, stel de sleutel handmatig in of geef `--allow-unconfigured` door.
    - Binden buiten loopback zonder authenticatie wordt geblokkeerd.
    - `--bind`-waarden `lan`, `tailnet` en `custom` worden momenteel via uitsluitend-IPv4-paden omgezet; uitsluitend-IPv6-configuraties met een eigen host vereisen een IPv4-sidecar of proxy vóór de Gateway.
    - `SIGUSR1` activeert na autorisatie een herstart binnen het proces. `commands.restart` (standaard: ingeschakeld) regelt extern verzonden `SIGUSR1`; stel dit in op `false` om handmatige herstarts via besturingssysteemsignalen te blokkeren. De agentgerichte tool `gateway` is alleen-lezen; agents vragen een herstart aan via de door een mens goedgekeurde delegatietool `openclaw`.
    - `SIGINT`/`SIGTERM` stoppen het proces, maar herstellen geen aangepaste terminalstatus — als je de CLI in een TUI of invoer in raw-modus verpakt, herstel je de terminal zelf vóór het afsluiten.

  </Accordion>
</AccordionGroup>

### Opties

<ParamField path="--port <port>" type="number">
  WebSocket-poort (standaard uit configuratie/omgeving; meestal `18789`).
</ParamField>
<ParamField path="--bind <mode>" type="string">
  Bindmodus: `loopback` (standaard), `lan`, `tailnet`, `auto`, `custom`.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gedeeld token voor `connect.params.auth.token`. Standaard `OPENCLAW_GATEWAY_TOKEN` wanneer dit is ingesteld.
</ParamField>
<ParamField path="--auth <mode>" type="string">
  Authenticatiemodus: `none`, `token`, `password`, `trusted-proxy`.
</ParamField>
<ParamField path="--password <password>" type="string">
  Wachtwoord voor `--auth password`.
</ParamField>
<ParamField path="--password-file <path>" type="string">
  Lees het Gateway-wachtwoord uit een bestand.
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  Tailscale-blootstelling: `off`, `serve`, `funnel`.
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  Stel de serve-/funnelconfiguratie van Tailscale bij het afsluiten opnieuw in.
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  Start zonder `gateway.mode=local` af te dwingen. Alleen voor ad-hoc-/ontwikkelbootstrap; configuratie wordt niet opgeslagen of gerepareerd.
</ParamField>
<ParamField path="--dev" type="boolean">
  Maak een ontwikkelconfiguratie en werkruimte als deze ontbreken (slaat `BOOTSTRAP.md` over).
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  Sta toe dat een ontwikkel-Gateway kanalen automatisch configureert vanuit omgevingsvariabelen in de omgeving. Vereist `--dev`.
</ParamField>
<ParamField path="--reset" type="boolean">
  Stel ontwikkelconfiguratie, inloggegevens, sessies en werkruimte opnieuw in. Vereist `--dev`.
</ParamField>
<ParamField path="--force" type="boolean">
  Beëindig vóór het starten elke bestaande listener op de doelpoort. In een niet-interactieve shell weigert dit een geverifieerde Gateway-listener te beëindigen; gebruik in plaats daarvan `--dev` of een geïsoleerde `--profile` met een vrije poort.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Uitgebreide logboekregistratie naar stdout/stderr.
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  Toon alleen logboeken van de CLI-backend in de console (schakelt ook stdout/stderr in).
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  WebSocket-logboekstijl: `auto`, `full`, `compact`.
</ParamField>
<ParamField path="--compact" type="boolean">
  Alias voor `--ws-log compact`.
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  Registreer onbewerkte modelstreamgebeurtenissen in JSONL.
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  JSONL-pad voor de onbewerkte stream.
</ParamField>

`--claude-cli-logs` is een verouderde alias voor `--cli-backend-logs`.

Stel voor `--bind custom` `gateway.customBindHost` in op een IPv4-adres. Elk ander adres dan `127.0.0.1` of `0.0.0.0` vereist op dezelfde poort ook `127.0.0.1` voor clients op dezelfde host; het opstarten mislukt als een van beide listeners niet kan binden. Jokerteken `0.0.0.0` voegt geen afzonderlijke vereiste alias toe. Uitsluitend-IPv6-configuraties met een eigen host vereisen een IPv4-sidecar of proxy vóór de Gateway.

## De Gateway opnieuw starten

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe` vraagt de actieve Gateway om actief werk vooraf te controleren en één samengevoegde herstart te plannen nadat dat werk is afgerond. De wachttijd is begrensd op 5 minuten; wanneer het tijdsbudget verstrijkt, wordt de herstart geforceerd. `--safe` kan niet worden gecombineerd met `--force` of `--wait`.

`--skip-deferral` omzeilt bij een veilige herstart de uitstelblokkering voor actief werk, zodat de Gateway onmiddellijk opnieuw wordt gestart, zelfs als er blokkeringen worden gemeld. Hiervoor is `--safe` vereist — gebruik dit wanneer uitstel vastzit door een onbeheersbare taak.

`--wait <duration>` overschrijft het budget voor het afronden van werk bij een gewone (niet-veilige) herstart. Accepteert milliseconden zonder eenheid of de eenheidsachtervoegsels `ms`, `s`, `m`, `h`, `d` (bijvoorbeeld `30s`, `5m`, `1h30m`); `--wait 0` wacht onbeperkt. Niet compatibel met `--force` of `--safe`.

`--force` slaat het afronden van actief werk over en start onmiddellijk opnieuw. Gewoon `restart` (zonder vlaggen) behoudt het bestaande herstartgedrag van de servicebeheerder.

<Warning>
Inline `--password` kan zichtbaar zijn in lokale proceslijsten. Geef de voorkeur aan `--password-file`, de omgeving of een door SecretRef ondersteunde `gateway.auth.password`.
</Warning>

### Externe supervisors

Stel `OPENCLAW_SUPERVISOR_MODE=external` alleen in wanneer een andere procesbeheerder eigenaar is van de Gateway-levenscyclus. In deze modus:

- `openclaw gateway restart` behoudt het bestaande veilige, geforceerde en begrensde wachtgedrag, maar richt zich op de geverifieerde actieve Gateway in plaats van launchd, systemd of Taakplanner.
- Bewerkingen voor het installeren, starten, stoppen en verwijderen van een systeemeigen service worden geweigerd, met de instructie om de externe supervisor te gebruiken.
- Zelfupdates van OpenClaw worden geweigerd, zodat de supervisor de Gateway kan stoppen, de runtime kan vervangen en voltooien en deze veilig opnieuw kan starten.
- Bij een herstart met een nieuw proces wordt vóór een nette afsluiting een begrensde SQLite-overdracht geschreven. Als opslag mislukt, valt de Gateway terug op een herstart binnen het proces in plaats van af te sluiten zonder een bruikbare overdracht.

`OPENCLAW_SERVICE_REPAIR_POLICY=external` blijft een afzonderlijk Doctor-reparatiebeleid. Het verklaart geen eigenaarschap van de runtime; supervisors die beide gedragingen nodig hebben, moeten beide variabelen instellen.

Externe supervisors kunnen via het verborgen machinecontract herstartoverdrachten onderhandelen en verwerken:

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

Protocolversie `1` ondersteunt de bewerking `consume`. Bij verwerking worden de verwachte PID en begrensde overdrachtsvelden binnen één onmiddellijke SQLite-transactie gevalideerd. Een geaccepteerde overdracht wordt verwijderd voordat succes wordt geretourneerd, zodat gelijktijdige of herhaalde verwerkers deze niet allebei kunnen accepteren. Een niet-overeenkomende PID wordt bewaard voor de overeenkomende eigenaar; ontbrekende, verlopen en ongeldige rijen geven geen toestemming voor een herstart.

Geldige machineverzoeken retourneren JSON met afsluitcode `0`, inclusief resultaten zonder herstart. Ongeldige argumenten retourneren `reason: "invalid-expected-pid"` met afsluitcode `2`; fouten in de statusopslag retourneren `reason: "store-unavailable"` met afsluitcode `1`. Supervisors moeten `capabilities` testen op exact de runtime of launcher die ze zullen gebruiken, in plaats van ondersteuning af te leiden uit een OpenClaw-versietekenreeks of het private SQLite-schema rechtstreeks te lezen.

### Gateway-profilering

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1` registreert fasetijden tijdens het opstarten, waaronder `eventLoopMax`-vertraging per fase en tijden van Plugin-opzoektabellen (installed-index, manifestregister, opstartplanning, owner-map-werk).
- `OPENCLAW_GATEWAY_RESTART_TRACE=1` registreert tot de herstart beperkte `restart trace:`-regels: signaalafhandeling, afronding van actief werk, afsluitfasen, volgende start, tijd tot gereedheid en geheugenstatistieken.
- `OPENCLAW_DIAGNOSTICS=timeline` met `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` schrijft naar beste vermogen een JSONL-tijdlijn met diagnostiek van het opstarten voor externe QA-harnassen (gelijkwaardig aan configuratie `diagnostics.flags: ["timeline"]`; het pad blijft uitsluitend via de omgeving instelbaar). Voeg `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` toe om event-loop-samples op te nemen.
- `pnpm build` en vervolgens `pnpm test:startup:gateway -- --runs 5 --warmup 1` benchmarken het opstarten van de Gateway aan de hand van het gebouwde CLI-invoerpunt: eerste procesuitvoer, `/healthz`, `/readyz`, tijdmetingen van de opstarttrace, event-loop-vertraging en timing van Plugin-opzoektabellen.
- `pnpm build` en vervolgens `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5` benchmarken een herstart binnen het proces op macOS of Linux (niet ondersteund op Windows; herstart vereist `SIGUSR1`). Gebruikt `SIGUSR1`, schakelt beide traces in het onderliggende proces in en registreert volgende `/healthz`, volgende `/readyz`, uitvaltijd, tijd tot gereedheid, CPU, RSS en herstarttracestatistieken.
- `/healthz` is levendheid; `/readyz` is bruikbare gereedheid. Behandel traceregels en benchmarkuitvoer als signalen voor toeschrijving aan de eigenaar, niet als een volledige prestatieconclusie op basis van één tijdspanne of sample.

## Een actieve Gateway opvragen

Alle opvraagopdrachten gebruiken WebSocket-RPC.

<Tabs>
  <Tab title="Uitvoermodi">
    - Standaard: leesbaar voor mensen (gekleurd in TTY).
    - `--json`: machineleesbare JSON (geen opmaak/spinner).
    - `--no-color` (of `NO_COLOR=1`): schakel ANSI uit met behoud van de menselijke lay-out.

  </Tab>
  <Tab title="Gedeelde opties">
    - `--url <url>`: WebSocket-URL van de Gateway.
    - `--token <token>`: Gateway-token.
    - `--password <password>`: Gateway-wachtwoord.
    - `--timeout <ms>`: time-out/budget (de standaard verschilt per opdracht; zie elke opdracht hieronder).
    - `--expect-final`: wacht op een 'definitief' antwoord (agentaanroepen).

  </Tab>
</Tabs>

<Note>
Wanneer je `--url` instelt, valt de CLI niet terug op inloggegevens uit de configuratie of omgeving. Geef `--token` of `--password` expliciet door. Ontbrekende expliciete inloggegevens zijn een fout.
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` is een livenessprobe: deze retourneert zodra de server HTTP kan beantwoorden. `/readyz` is strenger en blijft rood terwijl sidecars van opstartende plugins, kanalen of geconfigureerde hooks nog worden geïnitialiseerd. Lokale of geauthenticeerde gedetailleerde `/readyz`-antwoorden bevatten een diagnostisch `eventLoop`-blok (vertraging, benutting, verhouding tot CPU-kernen, `degraded`-vlag).

<ParamField path="--port <port>" type="number">
  Richt je op een lokale loopback-Gateway op deze poort. Overschrijft `OPENCLAW_GATEWAY_URL` en `OPENCLAW_GATEWAY_PORT` voor deze aanroep.
</ParamField>

### `gateway usage-cost`

Haal samenvattingen van gebruikskosten op uit sessielogboeken.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  Aantal op te nemen dagen.
</ParamField>
<ParamField path="--agent <id>" type="string">
  Beperk de samenvatting tot één geconfigureerde agent-id.
</ParamField>
<ParamField path="--all-agents" type="boolean">
  Aggregeer over alle geconfigureerde agents. Kan niet worden gecombineerd met `--agent`.
</ParamField>

### `gateway stability`

Haal de recente recorder voor diagnostische stabiliteit op uit een actieve Gateway.

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  Maximumaantal op te nemen recente gebeurtenissen (max. `1000`).
</ParamField>
<ParamField path="--type <type>" type="string">
  Filter op type diagnostische gebeurtenis, bijvoorbeeld `payload.large` of `diagnostic.memory.pressure`.
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  Neem alleen gebeurtenissen na een diagnostisch volgnummer op.
</ParamField>
<ParamField path="--bundle [path]" type="string">
  Lees een opgeslagen stabiliteitsbundel in plaats van de actieve Gateway aan te roepen. `--bundle latest` (of alleen `--bundle`) selecteert de nieuwste bundel in de statusmap; je kunt ook rechtstreeks een pad naar een bundel-JSON doorgeven.
</ParamField>
<ParamField path="--export" type="boolean">
  Schrijf een deelbaar ZIP-bestand met ondersteuningsdiagnostiek in plaats van stabiliteitsdetails af te drukken.
</ParamField>
<ParamField path="--output <path>" type="string">
  Uitvoerpad voor `--export`.
</ParamField>

<AccordionGroup>
  <Accordion title="Privacy en bundelgedrag">
    - Records bewaren operationele metagegevens: gebeurtenisnamen, aantallen, bytegroottes, geheugenmetingen, wachtrij-/sessiestatus, goedkeurings-id's, kanaal-/pluginnamen en geredigeerde sessiesamenvattingen. Ze sluiten chattekst, Webhook-bodies, tooluitvoer, onbewerkte request-/response-bodies, tokens, cookies, geheime waarden, hostnamen en onbewerkte sessie-id's uit. Stel `diagnostics.enabled: false` in om de recorder volledig uit te schakelen.
    - Fatale Gateway-afsluitingen, time-outs bij het afsluiten en opstartfouten na een herstart schrijven dezelfde diagnostische momentopname naar `~/.openclaw/logs/stability/openclaw-stability-*.json` wanneer de recorder gebeurtenissen bevat. Inspecteer de nieuwste bundel met `openclaw gateway stability --bundle latest`; `--limit`, `--type` en `--since-seq` zijn ook van toepassing op bundeluitvoer.

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

Schrijf een lokaal diagnostisch ZIP-bestand dat is ontworpen voor bugrapporten. Zie [Diagnostische export](/nl/gateway/diagnostics) voor het privacymodel en de inhoud van de bundel.

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  Pad voor het uitgevoerde ZIP-bestand. Standaard wordt een ondersteuningsexport in de statusmap gebruikt.
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  Maximumaantal op te nemen opgeschoonde logboekregels.
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  Maximumaantal te inspecteren logboekbytes.
</ParamField>
<ParamField path="--url <url>" type="string">
  WebSocket-URL van de Gateway voor de statusmomentopname.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway-token voor de statusmomentopname.
</ParamField>
<ParamField path="--password <password>" type="string">
  Gateway-wachtwoord voor de statusmomentopname.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  Time-out voor de status-/gezondheidsmomentopname.
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  Sla het zoeken naar een opgeslagen stabiliteitsbundel over.
</ParamField>
<ParamField path="--json" type="boolean">
  Druk het geschreven pad, de grootte en het manifest af als JSON.
</ParamField>

De export bundelt: `manifest.json` (bestandsinventaris), `summary.md` (Markdown-samenvatting), `diagnostics.json` (samenvatting op hoofdniveau van configuratie/logboeken/detectie/stabiliteit/status/gezondheid), `config/sanitized.json`, `status/gateway-status.json`, `health/gateway-health.json`, `logs/openclaw-sanitized.jsonl` en `stability/latest.json` wanneer er een bundel bestaat.

Deze is ontworpen om te worden gedeeld. De export bewaart operationele details die nuttig zijn voor foutopsporing — veilige logboekvelden, namen van subsystemen, statuscodes, tijdsduren, geconfigureerde modi, poorten, plugin-/provider-id's, niet-geheime functie-instellingen en geredigeerde operationele logboekberichten — en laat chattekst, Webhook-bodies, tooluitvoer, inloggegevens, cookies, account-/bericht-id's, prompt-/instructietekst, hostnamen en geheime waarden weg of redigeert deze. Wanneer een logboekbericht lijkt op payloadtekst van een gebruiker, chat of tool (bijvoorbeeld "gebruiker zei", "chattekst", "tooluitvoer", "Webhook-body"), bewaart de export alleen het feit dat een bericht is weggelaten, plus het aantal bytes ervan.

### `gateway status`

Toont de Gateway-service (launchd/systemd/schtasks), plus een optionele verbindings-/authenticatieprobe.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  Voeg een expliciet probedoel toe. De geconfigureerde externe host en localhost worden nog steeds geprobed.
</ParamField>
<ParamField path="--token <token>" type="string">
  Tokenauthenticatie voor de probe.
</ParamField>
<ParamField path="--password <password>" type="string">
  Wachtwoordauthenticatie voor de probe.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Time-out van de probe.
</ParamField>
<ParamField path="--no-probe" type="boolean">
  Sla de verbindingsprobe over (alleen serviceweergave).
</ParamField>
<ParamField path="--deep" type="boolean">
  Scan ook services op systeemniveau.
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  Breid de verbindingsprobe uit tot een leesprobe en sluit af met een niet-nulcode als deze mislukt. Kan niet worden gecombineerd met `--no-probe`.
</ParamField>

<AccordionGroup>
  <Accordion title="Statussemantiek">
    - Blijft beschikbaar voor diagnostiek, zelfs wanneer de lokale CLI-configuratie ontbreekt of ongeldig is.
    - De standaarduitvoer bewijst de servicestatus, WebSocket-verbinding en de authenticatiecapaciteit die tijdens de handshake zichtbaar is — niet lees-/schrijf-/beheerbewerkingen.
    - Probes wijzigen niets voor de eerste apparaatauthenticatie: ze hergebruiken een bestaand gecachet apparaattoken wanneer dat bestaat, maar maken nooit een nieuwe CLI-apparaatidentiteit of alleen-lezen-koppelingsrecord aan uitsluitend om de status te controleren.
    - Lost geconfigureerde SecretRefs voor probe-authenticatie waar mogelijk op. Als een vereiste SecretRef niet is opgelost, rapporteert `--json` `rpc.authWarning` wanneer de probe voor verbinding/authenticatie mislukt; geef `--token`/`--password` expliciet door of herstel de geheime bron. Waarschuwingen over niet-opgeloste authenticatie worden onderdrukt zodra de probe slaagt.
    - JSON-uitvoer bevat `gateway.version` wanneer de actieve Gateway dit rapporteert; `--require-rpc` kan terugvallen op de RPC-payload `status.runtimeVersion` als de handshakeprobe geen versiemetagegevens kan leveren.
    - Gebruik `--require-rpc` in scripts/automatisering wanneer een luisterende service niet voldoende is en RPC met leesbereik ook gezond moet zijn.
    - `--deep` scant op extra installaties van launchd/systemd/schtasks; wanneer meerdere Gateway-achtige services worden gevonden, drukt de voor mensen leesbare uitvoer opschoontips af (voer doorgaans één Gateway per machine uit) en rapporteert deze indien relevant een recente overdracht bij een herstart door de supervisor.
    - `--deep` voert ook configuratievalidatie uit in pluginbewuste modus (`pluginValidation: "full"`) en toont waarschuwingen uit pluginmanifesten (bijvoorbeeld ontbrekende metagegevens voor kanaalconfiguratie). De standaardwaarde `gateway status` behoudt het snelle alleen-lezen-pad dat pluginvalidatie overslaat.
    - De voor mensen leesbare uitvoer bevat het opgeloste pad van het bestandslogboek, plus de configuratiepaden en geldigheid van CLI versus service, om afwijkingen in het profiel of de statusmap te helpen diagnosticeren.
    - De voor mensen leesbare uitvoer bevat `Gateway heap:` met de toegepaste limiet en de adaptieve afleiding daarvan. JSON-uitvoer stelt hetzelfde rapport beschikbaar als `service.gatewayHeap`.

  </Accordion>
  <Accordion title="Controles op authenticatieafwijkingen in Linux systemd">
    - Controles op afwijkingen in service-authenticatie lezen zowel `Environment=` als `EnvironmentFile=` uit de unit (inclusief `%h`, paden tussen aanhalingstekens, meerdere bestanden en optionele `-`-bestanden).
    - Lost `gateway.auth.token`-SecretRefs op met behulp van de samengevoegde runtime-omgeving (eerst de omgeving van de serviceopdracht, daarna als terugval de procesomgeving).
    - Controles op tokenafwijkingen slaan het oplossen van het configuratietoken over wanneer tokenauthenticatie niet daadwerkelijk actief is (`gateway.auth.mode` expliciet `password`/`none`/`trusted-proxy`, of wanneer de modus niet is ingesteld en het wachtwoord kan prevaleren en geen tokenkandidaat kan prevaleren).

  </Accordion>
</AccordionGroup>

### `gateway probe`

De opdracht om "alles te debuggen". Deze probet altijd:

- je geconfigureerde externe Gateway (indien ingesteld), en
- localhost (loopback), **zelfs als een externe Gateway is geconfigureerd**.

Als je `--url` doorgeeft, wordt dat expliciete doel vóór beide toegevoegd. De voor mensen leesbare uitvoer labelt doelen als `URL (explicit)`, `Remote (configured)` / `Remote (configured, inactive)` en `Local loopback`.

<Note>
Als meerdere probedoelen bereikbaar zijn, worden ze allemaal afgedrukt. Een SSH-tunnel, TLS-/proxy-URL en geconfigureerde externe URL kunnen naar dezelfde Gateway verwijzen, zelfs met verschillende transportpoorten; `multiple_gateways` is gereserveerd voor afzonderlijke of qua identiteit ambigue bereikbare Gateways. Het uitvoeren van meerdere Gateways wordt ondersteund voor geïsoleerde profielen (bijvoorbeeld een herstelbot), maar de meeste installaties voeren één Gateway uit.
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  Gebruik deze poort voor het lokale loopback-probedoel en de externe poort van de SSH-tunnel. Zonder `--url` selecteert dit alleen het lokale loopback-doel in plaats van de geconfigureerde omgevings-URL van de Gateway, omgevingspoort of externe doelen.
</ParamField>

<AccordionGroup>
  <Accordion title="Interpretatie">
    - `Reachable: yes` betekent dat ten minste één doel een WebSocket-verbinding heeft geaccepteerd.
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` rapporteert wat de probe over authenticatie kon bewijzen, los van de bereikbaarheid.
    - `Read probe: ok` betekent dat RPC-detailaanroepen met leesbereik (`health`/`status`/`system-presence`/`config.get`) ook zijn geslaagd.
    - `Read probe: limited - missing scope: operator.read` betekent dat de verbinding is geslaagd, maar RPC met leesbereik beperkt is. Dit wordt gerapporteerd als **verminderde** bereikbaarheid, niet als een volledige mislukking.
    - `Read probe: failed` na `Connect: ok` betekent dat de WebSocket verbinding heeft gemaakt, maar dat daaropvolgende leesdiagnostiek een time-out kreeg of mislukte — eveneens **verminderd**, niet onbereikbaar.
    - Net als `gateway status` hergebruikt de probe bestaande gecachete apparaatauthenticatie, maar maakt deze geen apparaatidentiteit of koppelingsstatus voor het eerste gebruik aan.
    - De afsluitcode is alleen niet-nul wanneer geen enkel geprobed doel bereikbaar is.

  </Accordion>
  <Accordion title="JSON-uitvoer">
    Hoofdniveau:

    - `ok`: ten minste één doel is bereikbaar.
    - `degraded`: ten minste één doel heeft een verbinding geaccepteerd, maar heeft de volledige gedetailleerde RPC-diagnostiek niet voltooid.
    - `capability`: beste mogelijkheid die voor alle bereikbare doelen is waargenomen (`read_only`, `write_capable`, `admin_capable`, `pairing_pending`, `connected_no_operator_scope` of `unknown`).
    - `primaryTargetId`: beste doel om als actieve winnaar te behandelen, in deze volgorde: expliciete URL, SSH-tunnel, geconfigureerd extern doel, lokale loopback.
    - `warnings[]`: waarschuwingsrecords op basis van beste inspanning met `code`, `message`, optioneel `targetIds`.
    - `network`: hints voor lokale loopback-/tailnet-URL's, afgeleid van de huidige configuratie en het hostnetwerk.
    - `discovery.timeoutMs` / `discovery.count`: het daadwerkelijk gebruikte detectiebudget/aantal resultaten voor deze proberonde.

    Per doel (`targets[].connect`): `ok` (bereikbaarheid + classificatie als gedegradeerd), `rpcOk` (volledig geslaagde gedetailleerde RPC), `scopeLimited` (gedetailleerde RPC mislukt door ontbrekend operatorbereik).

    Per doel (`targets[].auth`): `role` en `scopes` gerapporteerd in `hello-ok` wanneer beschikbaar, plus de weergegeven classificatie `capability`.

  </Accordion>
  <Accordion title="Veelvoorkomende waarschuwingscodes">
    - `ssh_tunnel_failed`: het instellen van de SSH-tunnel is mislukt; de opdracht is teruggevallen op directe probes.
    - `multiple_gateways`: verschillende Gateway-identiteiten waren bereikbaar, of OpenClaw kon niet aantonen dat de bereikbare doelen dezelfde Gateway zijn. Een SSH-tunnel, proxy-URL of geconfigureerde externe URL naar dezelfde Gateway activeert dit niet.
    - `auth_secretref_unresolved`: een geconfigureerde SecretRef voor authenticatie kon voor een mislukt doel niet worden omgezet.
    - `probe_scope_limited`: de WebSocket-verbinding is geslaagd, maar de leesprobe werd beperkt door ontbrekende `operator.read`.
    - `local_tls_runtime_unavailable`: TLS voor de lokale Gateway is ingeschakeld, maar OpenClaw kon de vingerafdruk van het lokale certificaat niet laden.

  </Accordion>
</AccordionGroup>

#### Extern via SSH (gelijkwaardig aan de Mac-app)

De modus "Remote over SSH" van de macOS-app gebruikt lokale poortdoorschakeling, zodat een externe Gateway die alleen via loopback bereikbaar is, toegankelijk wordt op `ws://127.0.0.1:<port>`.

CLI-equivalent:

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` of `user@host:port` (poort is standaard `22`).
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  Identiteitsbestand.
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  Kies de eerste gedetecteerde Gateway-host als SSH-doel uit het omgezette detectie-eindpunt (`local.` plus het geconfigureerde wide-area-domein, indien aanwezig). Hints die alleen in TXT staan, worden genegeerd.
</ParamField>

Configuratiestandaarden (optioneel): `gateway.remote.sshTarget`, `gateway.remote.sshIdentity`.

### `gateway call <method>`

RPC-hulpprogramma op laag niveau.

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  JSON-objecttekenreeks voor parameters.
</ParamField>
<ParamField path="--url <url>" type="string">
  WebSocket-URL van de Gateway.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway-token.
</ParamField>
<ParamField path="--password <password>" type="string">
  Gateway-wachtwoord.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Time-outbudget.
</ParamField>
<ParamField path="--expect-final" type="boolean">
  Voornamelijk voor RPC's in agentstijl die tussentijdse gebeurtenissen streamen vóór een definitieve payload.
</ParamField>
<ParamField path="--json" type="boolean">
  Machineleesbare JSON-uitvoer.
</ParamField>

<Note>
`--params` moet geldige JSON zijn en elke methode valideert haar eigen parameterstructuur (extra of verkeerd benoemde velden worden geweigerd).
</Note>

## De Gateway-service beheren

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### Installeren met een wrapper

Gebruik `--wrapper` wanneer de beheerde service via een ander uitvoerbaar bestand moet starten, bijvoorbeeld een shim voor geheimenbeheer of een hulpprogramma om als een andere gebruiker uit te voeren. De wrapper ontvangt de normale Gateway-argumenten en is ervoor verantwoordelijk uiteindelijk `openclaw` of Node met die argumenten uit te voeren via exec.

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

Je kunt de wrapper ook via de omgeving instellen. `gateway install` controleert of het pad een uitvoerbaar bestand is, schrijft de wrapper naar de service-`ProgramArguments` en bewaart `OPENCLAW_WRAPPER` in de serviceomgeving voor latere gedwongen herinstallaties, updates en reparaties door doctor.

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

Als je een bewaarde wrapper wilt verwijderen, maak je `OPENCLAW_WRAPPER` leeg tijdens de herinstallatie:

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="Opdrachtopties">
    - `gateway status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
    - `gateway install`: `--port`, `--runtime <node>` (standaard: `node`), `--token`, `--wrapper <path>`, `--force`, `--json`
    - `gateway restart`: `--safe`, `--skip-deferral`, `--force`, `--wait <duration>`, `--json`
    - `gateway uninstall|start`: `--json`
    - `gateway stop`: `--disable`, `--force`, `--json`

  </Accordion>
  <Accordion title="Levenscyclusgedrag">
    - `gateway start` is idempotent: wanneer de beheerde service al actief is, rapporteert de opdracht het actieve proces en laat dit ongemoeid. Een geladen maar gestopte service wordt zoals voorheen gestart.
    - Gebruik `gateway restart` om een beheerde service opnieuw te starten. Koppel `gateway stop` en `gateway start` niet aan elkaar als vervanging voor opnieuw starten.
    - In een niet-interactieve shell vereist `gateway stop` de optie `--force`. Interactieve terminals behouden het bestaande gedrag zonder prompt. Geef voor automatisering en tests de voorkeur aan `gateway run --dev` of een geïsoleerde `--profile` met een vrije poort.
    - Op macOS gebruikt `gateway stop` standaard `launchctl bootout`, waarmee de LaunchAgent uit de huidige opstartsessie wordt verwijderd zonder een uitschakeling permanent te bewaren — automatisch herstel via KeepAlive blijft actief voor toekomstige crashes en `gateway start` schakelt de service weer correct in zonder een handmatige `launchctl enable`. Geef `--disable` door om KeepAlive en RunAtLoad permanent te onderdrukken, zodat de Gateway pas opnieuw wordt gestart na de volgende expliciete `gateway start`; gebruik dit wanneer een handmatige stop ook na opnieuw opstarten van het systeem moet blijven gelden.
    - Mutaties in de Gateway-levenscyclus voegen op basis van beste inspanning auditrecords met sleutel-waardeparen toe aan `<state-dir>/logs/gateway-restart.log`, waaronder start-, stop- en herstartbewerkingen via de CLI, veilige herstartverzoeken, herstarts door de supervisor en losgekoppelde overdrachten.
    - Levenscyclusopdrachten accepteren `--json` voor scripts.

  </Accordion>
  <Accordion title="Heapgrootte van de beheerde Gateway">
    - `gateway install` schrijft een uitsluitend voor de heap bestemde waarde voor `NODE_OPTIONS` voor de beheerde Gateway-service. De waarde richt zich op 50% van het beperkte geheugen wanneer Node een container- of servicelimiet rapporteert, en anders op 50% van het fysieke geheugen.
    - Het nominale doelbereik is 2048–8192 MiB, met een aanvullende limiet die 75% ruimte voor native geheugen vrijhoudt. Op kleine hosts kan deze limiet ervoor zorgen dat de toegepaste limiet onder de nominale ondergrens van 2048 MiB ligt.
    - Een geldige expliciete `--max-old-space-size` die al in de geïnstalleerde service is opgeslagen, blijft behouden bij gedwongen herinstallaties en reparaties door doctor. Andere `NODE_OPTIONS`-vlaggen worden niet overgenomen in de beheerde service.
    - `NODE_OPTIONS` uit de omringende shell overschrijft dit beleid niet. Gebruik `gateway status` of `doctor` om de geïnstalleerde waarde te inspecteren; voer `openclaw gateway install --force` uit om oudere servicemetadata zonder beheerde heapinstelling opnieuw te genereren.
    - Het beleid geldt alleen voor de beheerde Gateway-service. `gateway run` op de voorgrond, Node-services en handmatig geschreven supervisoreenheden behouden hun eigen runtimeconfiguratie.

  </Accordion>
  <Accordion title="Authenticatie en SecretRefs tijdens de installatie">
    - Wanneer tokenauthenticatie een token vereist en `gateway.auth.token` door SecretRef wordt beheerd, controleert `gateway install` of de SecretRef kan worden omgezet, maar wordt het omgezette token niet opgeslagen in de omgevingsmetadata van de service.
    - Als tokenauthenticatie een token vereist en de geconfigureerde SecretRef voor het token niet kan worden omgezet, wordt de installatie veilig geblokkeerd in plaats van terug te vallen op het opslaan van platte tekst.
    - Geef voor wachtwoordauthenticatie op `gateway run` de voorkeur aan `OPENCLAW_GATEWAY_PASSWORD`, `--password-file` of een door SecretRef ondersteunde `gateway.auth.password` boven een inline `--password`.
    - In de afgeleide authenticatiemodus versoepelt `OPENCLAW_GATEWAY_PASSWORD` die alleen in de shell is ingesteld de tokenvereisten voor installatie niet; gebruik duurzame configuratie (`gateway.auth.password` of configuratie-`env`) wanneer je een beheerde service installeert.
    - Als zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd en `gateway.auth.mode` niet is ingesteld, wordt de installatie geblokkeerd totdat de modus expliciet is ingesteld.

  </Accordion>
</AccordionGroup>

## Gateways detecteren (Bonjour)

`gateway discover` scant naar Gateway-bakens (`_openclaw-gw._tcp`).

- Multicast DNS-SD: `local.`
- Unicast DNS-SD (wide-area Bonjour): kies een domein (bijvoorbeeld `openclaw.internal.`) en stel split DNS plus een DNS-server in; zie [Bonjour](/nl/gateway/bonjour).

Alleen Gateways waarvoor Bonjour-detectie is ingeschakeld (standaard) adverteren het baken.

TXT-hints op elk baken: `role` (hint voor Gateway-rol), `transport` (transporthint, bijvoorbeeld `gateway`), `gatewayPort` (WebSocket-poort, meestal `18789`), `tailnetDns` (MagicDNS-hostnaam, indien beschikbaar), `gatewayTls` / `gatewayTlsSha256` (TLS ingeschakeld + certificaatvingerafdruk). `sshPort` en `cliPath` worden alleen gepubliceerd in de volledige detectiemodus (`discovery.mdns.mode: "full"`; standaard is `"minimal"`, waarin ze worden weggelaten — clients gebruiken dan standaard poort `22` voor SSH-doelen).

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  Time-out per opdracht (bladeren/omzetten).
</ParamField>
<ParamField path="--json" type="boolean">
  Machineleesbare uitvoer (schakelt ook opmaak/spinner uit).
</ParamField>

Voorbeelden:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- Scant `local.` plus het geconfigureerde wide-area-domein wanneer dit is ingeschakeld.
- `wsUrl` in JSON-uitvoer wordt afgeleid van het omgezette service-eindpunt, niet van hints die alleen in TXT staan, zoals `lanHost` of `tailnetDns`.
- `discovery.mdns.mode` bepaalt de publicatie van `sshPort`/`cliPath` op zowel `local.` mDNS als wide-area DNS-SD (zie hierboven).

</Note>

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Gateway-runbook](/nl/gateway)
