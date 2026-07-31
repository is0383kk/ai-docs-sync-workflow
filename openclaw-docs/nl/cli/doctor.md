---
read_when:
    - Je hebt verbindings-/authenticatieproblemen en wilt begeleide oplossingen
    - Je hebt bijgewerkt en wilt een snelle controle.
summary: CLI-referentie voor `openclaw doctor` (statuscontroles + begeleide reparaties)
title: Diagnosehulpmiddel
x-i18n:
    generated_at: "2026-07-27T04:53:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2b0aa9b51d7bccd4357d3ec747be514a0245b44a90e6e6c7ea789ab68420465
    source_path: cli/doctor.md
    workflow: 16
---

# `openclaw doctor`

Gezondheidscontroles en snelle oplossingen voor de Gateway, kanalen, plugins, Skills, modelroutering, lokale status en configuratiemigraties. Gebruik dit wanneer iets niet werkt zoals verwacht en je met één opdracht wilt achterhalen wat er mis is.

Wanneer de Gateway-status gedegradeerde SecretRef-eigenaren meldt, toont doctor een waarschuwing **Degradatie van de secretruntime** met elke koude of verouderde eigenaar, het betreffende configuratiepad, de geredigeerde reden en de opdracht `openclaw secrets reload` om het opnieuw te proberen.

Wanneer binnenkomende kanaalgebeurtenissen naar de dead-letter-wachtrij worden verplaatst, noemt doctor elk betrokken kanaalaccount en verwijst het voor inspectie en herstel naar [`openclaw channels dead-letters list`](/nl/cli/channels#inbound-dead-letters).

Gerelateerd:

- Probleemoplossing: [Probleemoplossing](/nl/gateway/troubleshooting)
- Beveiligingsaudit: [Beveiliging](/nl/gateway/security)

## Modi

Doctor heeft vijf modi:

| Modus                     | Opdracht                                  | Gedrag                                                                                   |
| ------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------- |
| Inspectie                 | `openclaw doctor`                         | Mensgerichte controles en begeleide prompts.                                             |
| Reparatie                 | `openclaw doctor --fix`                   | Voert ondersteunde reparaties uit en gebruikt prompts, tenzij niet-interactieve reparatie veilig is. |
| Lint                      | `openclaw doctor --lint`                  | Alleen-lezen gestructureerde bevindingen voor CI, preflightcontroles en reviewpoorten.    |
| Gedeeld SQLite-onderhoud  | `openclaw doctor --state-sqlite compact`  | Voert expliciet een checkpoint, compactie en verificatie uit op de canonieke gedeelde statusdatabase. |
| SQLite-sessiemigratie     | `openclaw doctor --session-sqlite <mode>` | Inspecteert, importeert, valideert, compacteert, herstelt of zet sessiestatus terug.      |

Gebruik bij voorkeur `--lint` wanneer automatisering een stabiel resultaat nodig heeft. Gebruik bij voorkeur `--fix` wanneer een menselijke operator wil dat doctor de configuratie of status bewerkt.

## Voorbeelden

```bash
openclaw doctor
openclaw doctor --lint
openclaw doctor --lint --json
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --deep
openclaw doctor --fix
openclaw doctor --fix --non-interactive
openclaw doctor --generate-gateway-token
openclaw doctor --post-upgrade
openclaw doctor --post-upgrade --json
openclaw doctor --state-sqlite compact
openclaw doctor --state-sqlite compact --json
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-agent main --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Gebruik voor kanaalspecifieke machtigingen de kanaalprobes in plaats van `doctor`:

```bash
openclaw channels capabilities --channel discord --target channel:<channel-id>
openclaw channels status --probe
```

`channels capabilities` meldt de effectieve machtigingen van de bot voor een specifiek kanaaldoel. `channels status --probe` controleert alle geconfigureerde kanalen en doelen voor automatisch deelnemen aan spraakkanalen.

## Opties

| Optie                           | Effect                                                                                                                                                                                  |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-workspace-suggestions`    | Schakelt suggesties voor werkruimtegeheugen en zoeken uit.                                                                                                                              |
| `--yes`                         | Accepteert standaardwaarden zonder prompt.                                                                                                                                              |
| `--repair` / `--fix`            | Voert aanbevolen reparaties buiten services uit zonder prompt (`--fix` is een alias). Installaties en herschrijvingen van de Gateway-service vereisen nog steeds interactieve bevestiging of expliciete `gateway`-opdrachten. |
| `--force`                       | Voert ingrijpende reparaties uit, waaronder het overschrijven van aangepaste serviceconfiguratie.                                                                                       |
| `--non-interactive`             | Wordt zonder prompts uitgevoerd; alleen veilige migraties en reparaties buiten services.                                                                                                |
| `--generate-gateway-token`      | Genereert en configureert een Gateway-token.                                                                                                                                            |
| `--allow-exec`                  | Staat doctor toe geconfigureerde `exec`-SecretRefs uit te voeren tijdens het verifiëren van secrets.                                                                         |
| `--deep`                        | Scant systeemservices op extra Gateway-installaties en meldt recente overdrachten van herstarts door de Gateway-supervisor.                                                             |
| `--lint`                        | Voert gemoderniseerde gezondheidscontroles uit in alleen-lezenmodus en geeft diagnostische bevindingen weer.                                                                            |
| `--post-upgrade`                | Voert na een upgrade compatibiliteitsprobes voor plugins uit; bevindingen gaan naar stdout; afsluitcode 1 als er een bevinding op foutniveau aanwezig is.                                |
| `--state-sqlite <mode>`         | Voert expliciet SQLite-onderhoud voor de gedeelde status uit. De enige modus is `compact`.                                                                                      |
| `--session-sqlite <mode>`       | Voert de gerichte SQLite-sessiemigratiemodus uit: `inspect`, `dry-run`, `import`, `validate`, `compact`, `recover` of `restore`. |
| `--session-sqlite-store <path>` | Met `--session-sqlite`: selecteert één verouderd `sessions.json`-opslagpad.                                                                                                           |
| `--session-sqlite-agent <id>`   | Met `--session-sqlite`: selecteert één geconfigureerde agent.                                                                                                                            |
| `--session-sqlite-all-agents`   | Met `--session-sqlite`: selecteert geconfigureerde en gedetecteerde agentopslaglocaties.                                                                                                 |
| `--github-issue`                | Met `--session-sqlite recover`: bereidt een geschoond probleemrapport voor openclaw/openclaw voor; doctor maakt dit met `gh` na `--yes` of interactieve bevestiging.     |
| `--json`                        | Met `--lint`: JSON-bevindingen. Met `--post-upgrade`: `{ probesRun, findings }`. Met `--state-sqlite` of `--session-sqlite`: het onderhoudsrapport als JSON.                        |
| `--severity-min <level>`        | Met `--lint`: laat bevindingen onder `info`, `warning` of `error` weg.                                                                          |
| `--all`                         | Met `--lint`: voert alle geregistreerde controles uit, inclusief optionele controles die van de standaardset zijn uitgesloten.                                                |
| `--skip <id>`                   | Met `--lint`: slaat een controle-id over. Herhaalbaar.                                                                                                                         |
| `--only <id>`                   | Met `--lint`: voert alleen de opgegeven controle-id('s) uit. Herhaalbaar.                                                                                                      |

`--severity-min`, `--all`, `--only` en `--skip` worden alleen samen met `--lint` geaccepteerd; `--json` wordt geaccepteerd met `--lint`, `--post-upgrade`, `--state-sqlite` en `--session-sqlite`.

## Lintmodus

`openclaw doctor --lint` is alleen-lezen: geen prompts, geen reparatie en geen herschrijvingen van configuratie of status.

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

Voor mensen leesbare uitvoer is compact:

```text
doctor --lint: 6 controle(s) uitgevoerd, 1 bevinding(en)
  [warning] core/doctor/gateway-config gateway.mode - gateway.mode is niet ingesteld; het starten van de Gateway wordt geblokkeerd.
    oplossing: Voer `openclaw configure` uit en stel de Gateway-modus (local/remote) in, of voer `openclaw config set gateway.mode local` uit.
```

JSON-uitvoer is de interface voor scripts:

```json
{
  "ok": false,
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": [
    {
      "checkId": "core/doctor/gateway-config",
      "severity": "warning",
      "message": "gateway.mode is niet ingesteld; het starten van de Gateway wordt geblokkeerd.",
      "path": "gateway.mode",
      "fixHint": "Voer `openclaw configure` uit en stel de Gateway-modus (local/remote) in, of voer `openclaw config set gateway.mode local` uit."
    }
  ]
}
```

Afsluitcodes:

| Code | Betekenis                                                                 |
| ---- | ------------------------------------------------------------------------- |
| `0`  | Geen bevindingen op of boven de geselecteerde ernstgrens.                 |
| `1`  | Minstens één bevinding voldoet aan de geselecteerde grens.                |
| `2`  | Opdracht- of runtimefout voordat lintbevindingen kunnen worden geproduceerd. |

`--severity-min` bepaalt zowel welke bevindingen worden weergegeven als de afsluitgrens: `openclaw doctor --lint --severity-min error` kan niets weergeven en afsluiten met `0`, zelfs wanneer bevindingen met een lagere ernstgraad van `info`/`warning` bestaan.

`--all` bepaalt welke controles vóór de ernstfiltering worden geselecteerd. De standaard-lintuitvoering sluit controles uit die diepgaand of historisch zijn, of die eerder herstelbare verouderde restanten aan het licht brengen; gebruik `--all` voor de volledige inventaris. `--only <id>` is de nauwkeurigste selector en kan elke geregistreerde controle op id uitvoeren.

`core/doctor/local-audio-acceleration` meldt de automatisch geselecteerde lokale STT-opdracht, afzonderlijk bewijs voor geschikte, aangevraagde en waargenomen backends, en de terugvalvolgorde zonder een spraakmodel te laden. Dit levert een informatieve bevinding op; neem daarom `--severity-min info` op om deze weer te geven.

## Gestructureerde gezondheidscontroles

Moderne doctor-controles gebruiken een klein opgesplitst contract:

```ts
detect(ctx, scope?) -> HealthFinding[]
repair?(ctx, findings) -> HealthRepairResult
```

`detect()` vormt de basis voor `doctor --lint`. `repair()` is optioneel en wordt alleen uitgevoerd onder `doctor --fix` / `doctor --repair`. Controles die nog niet naar deze vorm zijn gemigreerd, gebruiken nog steeds de verouderde doctor-bijdrageflow.

Herstelcontexten kunnen `dryRun`- en `diff`-verzoeken bevatten; herstelresultaten kunnen gestructureerde `diffs` (configuratie-/bestandsbewerkingen) en `effects` (neveneffecten voor services, processen, pakketten, status of andere onderdelen) retourneren, zodat geconverteerde controles kunnen doorgroeien naar `doctor --fix --dry-run` zonder de planning van wijzigingen naar `detect()` te verplaatsen.

`repair()` rapporteert `status: "repaired" | "skipped" | "failed"` (een weggelaten status betekent `repaired`). Wanneer herstel `skipped` of `failed` retourneert, rapporteert Doctor de reden en slaat het de validatie voor die controle over. Na een geslaagd herstel voert Doctor `detect()` opnieuw uit, beperkt tot de herstelde bevindingen; als de bevinding nog steeds aanwezig is, rapporteert Doctor een herstelwaarschuwing in plaats van de wijziging als voltooid te beschouwen.

Een bevinding bevat:

| Veld              | Doel                                                   |
| ----------------- | ------------------------------------------------------ |
| `checkId`         | Stabiele id voor skip/only-filters en CI-toelatingslijsten. |
| `severity`        | `info`, `warning` of `error`.                         |
| `message`         | Voor mensen leesbare probleembeschrijving.             |
| `path`            | Configuratie-, bestands- of logisch pad indien beschikbaar. |
| `line` / `column` | Bronlocatie indien beschikbaar.                        |
| `ocPath`          | Nauwkeurig `oc://`-adres wanneer een controle ernaar kan verwijzen. |
| `fixHint`         | Voorgestelde operatoractie of herstelsamenvatting.     |

Gemoderniseerde controles van de kern-Doctor blijven gekoppeld aan de geordende Doctor-bijdrage die eigenaar is van hun menselijke `doctor`- / `doctor --fix`-gedrag. Het gedeelde gestructureerde gezondheidsregister is het uitbreidingspunt: gebundelde en door plugins ondersteunde controles worden na de controles van de kern-Doctor uitgevoerd zodra het pakket dat er eigenaar van is deze registreert in het actieve opdrachtpad. `openclaw/plugin-sdk/health` stelt hetzelfde contract beschikbaar aan auteurs van plugins.

## Controles selecteren

```bash
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --skip core/doctor/skills-readiness
openclaw doctor --lint --all --skip core/doctor/session-locks
```

`--only` en `--skip` accepteren volledige controle-id's en mogen worden herhaald. Als een `--only`-id niet is geregistreerd, wordt voor die id geen controle uitgevoerd; gebruik `checksRun`/`checksSkipped` in de uitvoer om te bevestigen dat een gerichte gate de verwachte controles selecteert.

## Modus na upgrade

`openclaw doctor --post-upgrade` voert compatibiliteitsproeven voor plugins uit om na een build of upgrade aaneen te schakelen. Bevindingen gaan naar stdout; de afsluitcode is 1 als een bevinding `level: "error"` heeft. Voeg `--json` toe voor een machineleesbare envelop (`{ probesRun, findings }`), geschikt voor CI, de communityskill `fork-upgrade` en andere smoketesttools voor na een upgrade. Als de index van geïnstalleerde plugins ontbreekt of ongeldig is, geeft de JSON-modus nog steeds de envelop uit met een `plugin.index_unavailable`-foutbevinding.

Het opstarten van een containerimage is de uitzondering op de gebruikelijke stroom 'Doctor uitvoeren na
bijwerken'. Wanneer `openclaw gateway run` op een nieuwe OpenClaw-versie start, voert het
veilige herstelbewerkingen voor status en plugins uit voordat het gereedheid rapporteert. Als herstel niet
veilig kan worden voltooid, stopt het opstartproces en krijg je de instructie om dezelfde image eenmaal uit te voeren met
`openclaw doctor --fix` voor dezelfde gekoppelde status/configuratie voordat je
de container normaal opnieuw start.

## Migratie van verouderde status

`openclaw doctor --fix` is de enige eigenaar van permanente migraties van bestanden naar SQLite. Het valideert en claimt elke herkende bron, schrijft en verifieert canonieke rijen, legt een migratiebewijs vast en verwijdert vervolgens de buiten gebruik gestelde bron. Runtimecode voert geen luie imports of fallbacklezingen uit.

Dit omvat buiten gebruik gestelde MCP OAuth-bestanden onder `<state-dir>/mcp-oauth/*.json`. Stop de Gateway vóór herstel. Doctor importeert geldige inloggegevens in `<state-dir>/state/openclaw.sqlite`, behoudt een bestaande canonieke SQLite-sessie wanneer beide opslagplaatsen bestaan, verwijdert de verouderde opgeslagen OAuth-waarde `state` en gebruikt het bewijs om te voorkomen dat een opnieuw aangemaakt verouderd bestand uitgelogde inloggegevens opnieuw activeert. Buiten gebruik gestelde `.lock`-sidecars worden gesloten bij fouten: als Doctor een verouderde eigenaar rapporteert, controleer dan of er geen ouder OpenClaw-proces actief is, verwijder die sidecar en voer Doctor opnieuw uit.

## Compaction van gedeelde status in SQLite

Zie [Databaseschema's](/nl/reference/database-schemas) voor schemaversiebeheer, integriteitscontroles en herstel na een downgrade.

`openclaw doctor --state-sqlite compact` is expliciet offline onderhoud voor
de canonieke database met gedeelde status op
`<state-dir>/state/openclaw.sqlite`. Het accepteert geen willekeurig databasepad,
wordt nooit aangeroepen door de normale werking van de Gateway en maakt geen deel uit van
`openclaw doctor --fix`. De opdracht verkrijgt dezelfde eigendomsvergrendeling voor status als
bij het opstarten van de Gateway en houdt deze vast tijdens validatie, checkpointing, `VACUUM` en
de laatste integriteitscontroles. De opdracht weigert te worden uitgevoerd zolang een Gateway of een andere
SQLite-onderhoudsopdracht eigenaar is van die vergrendeling. De statusvergrendeling blijft actief wanneer
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` de Gateway-singleton per configuratie overslaat, zodat een
operatorshell de omgeving van de Gateway-service niet hoeft over te nemen om
deze tijdens onderhoud te detecteren.

Stop de Gateway en maak eerst een geverifieerde back-up:

```bash
openclaw gateway stop
openclaw backup create --verify
openclaw doctor --state-sqlite compact --json
openclaw gateway start
```

De opdracht:

1. Vereist een regulier bestand op het canonieke pad voor gedeelde status. Een ontbrekende
   database wordt gerapporteerd als `skipped` en de opdracht wordt succesvol afgesloten.
2. Valideert de huidige ondersteunde schemaversie en
   `schema_meta.role = "global"` voordat een checkpoint wordt gemaakt of het bestand wordt gewijzigd.
3. Vereist een niet-bezette `wal_checkpoint(TRUNCATE)`. Stop alle resterende OpenClaw-
   processen en probeer het opnieuw als het checkpoint bezet is.
4. Stelt `auto_vacuum` in op `INCREMENTAL`, voert een volledige `VACUUM` uit en maakt
   opnieuw een checkpoint.
5. Voert `quick_check`, `integrity_check` en `foreign_key_check` uit en
   past vervolgens opnieuw uitsluitend-eigenaarmachtigingen toe op de database en SQLite-sidecarbestanden.

JSON-uitvoer rapporteert vóór en na Compaction de grootte van de database en WAL,
freelist-pagina's, paginagrootte en de waarde
`auto_vacuum`, plus het aantal teruggewonnen bytes en de resultaten van
`quick_check` en `integrity_check`. `foreign_key_check` wordt
gesloten bij fouten afgedwongen en heeft geen afzonderlijk succesveld. SQLite rapporteert `auto_vacuum` als
`0` voor geen, `1` voor volledig en `2` voor incrementeel.

Compaction mislukt zonder wijzigingen wanneer het schema oud is, nieuwer is dan de
actieve OpenClaw-build of bij een agentdatabase hoort. Voer voor een ouder schema van gedeelde status
eerst `openclaw doctor --fix` uit. Herstel een
compatibele back-up of upgrade OpenClaw voor een nieuwer schema.

## SQLite-migratie van sessies

OpenClaw importeert verouderde sessierijen en transcriptgeschiedenis automatisch in de SQLite-database van elke agent
tijdens het opstarten van de Gateway en tijdens
`openclaw doctor --fix`. `openclaw doctor --session-sqlite <mode>` is het
gerichte hulpmiddel voor inspectie en validatie van die migratie. Actuele runtimesessierijen
bevinden zich in
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. Verouderde
`sessions.json`-bestanden zijn migratiebronnen. Actieve JSONL-transcriptbestanden worden
geïmporteerd en na een geslaagde import uit de actieve sessiemap gearchiveerd;
JSONL-bestanden in de archieflaag blijven ondersteuningsartefacten, geen runtime-
fallbacks.

Modi:

| Modus      | Gedrag                                                                                                                 |
| ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inspect`  | Leest aantallen uit verouderde bronnen en SQLite, plus JSONL-bestanden zonder verwijzing, zonder te importeren.        |
| `dry-run`  | Parseert verouderde vermeldingen en JSONL-transcriptbestanden, telt importeerbare rijen en rapporteert problemen zonder SQLite-rijen te schrijven. |
| `import`   | Importeert verouderde vermeldingen en transcriptgebeurtenissen in SQLite voor de geselecteerde doelen.                |
| `validate` | Vergelijkt de geselecteerde verouderde bronnen met SQLite-rijen en aantallen transcriptgebeurtenissen.                  |
| `compact`  | Maakt checkpoints en voert VACUUM uit op geselecteerde SQLite-agentdatabases om vrije pagina's terug te winnen na grote verwijderingen of het opschonen van archieven. |
| `recover`  | Herstelt de laatst mislukte migratie-uitvoering, valideert de doelen ervan en stelt een opgeschoond GitHub-issuereport op. |
| `restore`  | Herstelt gearchiveerde transcriptartefacten vanuit vastgelegde migratiemanifesten zonder SQLite-gegevens te verwijderen. |

Selectors:

- Standaard: de geconfigureerde opslag van de standaardagent, wanneer dat verouderde opslagbestand bestaat.
- `--session-sqlite-agent <id>`: één geconfigureerde agent.
- `--session-sqlite-all-agents`: geconfigureerde agentopslagplaatsen plus ontdekte agentopslagplaatsen.
- `--session-sqlite-store <path>`: één expliciet pad naar een verouderde `sessions.json`.

Handmatige inspectiereeks:

```bash
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-all-agents --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
```

Maak een back-up van de OpenClaw-statusmap voordat je `import` uitvoert op een installatie met
belangrijke geschiedenis. `validate` wordt afgesloten met een niet-nulcode wanneer een geselecteerde verouderde vermelding
ontbreekt in SQLite, een sessie-id afwijkt of het aantal transcriptgebeurtenissen afwijkt.
Controleer bij gebruik van `--session-sqlite-store <path>` of het rapport het
verwachte aantal doelen bevat; een niet-bestaand expliciet opslagpad selecteert geen doelen.

SQLite-verwijderingen winnen eerst pagina's binnen de database terug; ze verkleinen
het databasebestand niet noodzakelijk onmiddellijk. Voer na het verwijderen of archiveren van grote
transcripten `openclaw doctor --session-sqlite compact --session-sqlite-all-agents` uit
om checkpoints voor WAL-bestanden te maken, `VACUUM` uit te voeren en de grootte van database en WAL
vóór en na de bewerking te rapporteren. Compaction vereist een regulier bestand met het huidige agentschema, de
duurzame eigenaarsmetadata van de geselecteerde agent en geen open handle in het Doctor-
proces. De destructieve modi `import`, `compact`, `recover` en `restore`
houden gedurende hun volledige werking dezelfde eigendomsvergrendeling voor status vast als bij het opstarten van de Gateway;
`inspect`, `dry-run` en `validate` blijven alleen-lezen en verkrijgen deze niet. Stop
eerst de Gateway. Destructieve modi mislukken in plaats van te wedijveren met actieve schrijfbewerkingen of
een andere onderhoudsopdracht. Een destructief `--session-sqlite-store`-
doel moet zich in de actieve statusmap bevinden; stel `OPENCLAW_STATE_DIR` in op
de statusmap die eigenaar is van de opslag voordat je een andere installatie onderhoudt.
Bestaande hardgekoppelde doelen worden geweigerd omdat een ander pad
dezelfde database-inode buiten de vergrendelde statusmap kan delen. Dezelfde eigendomscontroles
gelden voor SQLite-WAL-, gedeeld-geheugen- en rollback-journal-sidecars.

Elke import schrijft een manifest onder
`~/.openclaw/session-sqlite-migration-runs/` voordat transcriptartefacten
naar het archief worden verplaatst. Als het opstarten een mislukte SQLite-sessiemigratie rapporteert nadat
artefacten zijn verplaatst, voer je herstel uit:

```bash
openclaw doctor --session-sqlite recover --github-issue
```

Herstel selecteert het nieuwste manifest van een mislukte migratie, herstelt alleen de
gearchiveerde artefacten van het manifest, valideert de betrokken doelen, vernieuwt de
opgeschoonde rapporten `.failure.md` en `.failure.json` en bereidt de hoofdtekst
van een GitHub-issue voor zonder transcriptinhoud, onbewerkte omgevingsgegevens, geheimen
en onbegrensde configuratie. Als er geen manifest van een mislukte migratie bestaat, maar
een geselecteerde SQLite-database van een agent beschadigd of geen database is, of
journaal-sidecars zonder hoofddatabase bevat, kopieert herstel de volledige bestandenset
naar een tijdelijke inspectiemap. SQLite kan een geldig actief journaal in die tijdelijke
kopie terugdraaien voordat `quick_check`, `integrity_check` en `foreign_key_check`
worden uitgevoerd, terwijl de oorspronkelijke forensische bestanden onaangeroerd blijven.
Bij mislukte integriteitscontroles of verweesde sidecars worden de DB-, WAL-, SHM- en
rollbackjournaalbestanden behouden door de volledige aangetroffen set te hernoemen met één
achtervoegsel `.corrupt-<timestamp>`. Bij een opgevangen fout tijdens het hernoemen worden
reeds verplaatste bestanden teruggezet voordat de fout wordt gemeld, zodat een herstelbare
bestandenset niet ongemerkt wordt opgesplitst. Stop de Gateway vóór herstel; het kopiëren
of hernoemen van een actief veranderende SQLite-bestandenset is onveilig en gedraagt zich
verschillend per besturingssysteem. Met `--github-issue --yes` gebruikt doctor de GitHub CLI
om het issue in `openclaw/openclaw` aan te maken; zonder bevestiging schrijft doctor het
lokale ondersteuningsrapport en toont het een vooraf ingevulde issue-URL.

`restore` blijft de onderliggende bewerking voor ongedaan maken. Deze gebruikt
`sourcePath -> archivePath`-records uit het manifest, verplaatst gearchiveerde artefacten alleen
terug als het oorspronkelijke pad ontbreekt, meldt conflicten wanneer beide paden bestaan
en laat de SQLite-database staan.

### Downgraden na de SQLite-migratie van sessies

Herstel de gearchiveerde verouderde transcriptartefacten voordat je een oudere,
bestandsgebaseerde versie van OpenClaw start:

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Oudere versies lezen `sessions.json`-vermeldingen en de in die vermeldingen vastgelegde
`sessionFile`-paden. Na de SQLite-migratie verplaatsen geslaagde imports actieve
JSONL-transcripten naar `session-sqlite-import-archive/`, waardoor de oudere runtime die geschiedenis
pas kan zien nadat herstel de in het manifest vastgelegde artefacten naar hun
oorspronkelijke paden heeft teruggezet.

Herstel verwijdert geen SQLite-gegevens. Sessies die na de overstap naar SQLite zijn
aangemaakt, bestaan alleen in SQLite en zijn niet zichtbaar voor de oudere runtime. Als
je later opnieuw upgradet, voer dan de normale bovenstaande validatiereeks voor migratie
uit, zodat OpenClaw de herstelde verouderde artefacten met de SQLite-rijen kan vergelijken
voordat ze worden geïmporteerd.

## Opmerkingen

- In Nix-modus (`OPENCLAW_NIX_MODE=1`) werken alleen-lezencontroles van doctor nog steeds, maar `doctor --fix`, `doctor --repair`, `doctor --yes` en `doctor --generate-gateway-token` zijn uitgeschakeld omdat `openclaw.json` onveranderlijk is. Bewerk in plaats daarvan de Nix-bron voor deze installatie; gebruik voor nix-openclaw de agentgerichte [Snelstart](https://github.com/openclaw/nix-openclaw#quick-start).
- Interactieve prompts (oplossingen voor sleutelhangertoegang/OAuth enzovoort) worden alleen uitgevoerd wanneer stdin een TTY is en `--non-interactive` **niet** is ingesteld. Headless-uitvoeringen (cron, Telegram, geen terminal) slaan prompts over.
- Niet-interactieve uitvoeringen van `doctor` slaan het vroegtijdig laden van plugins over, zodat headless-statuscontroles snel blijven. Interactieve sessies laden nog steeds de pluginoppervlakken die nodig zijn voor de verouderde status-/reparatiestroom.
- `--lint` is strenger dan `--non-interactive`: altijd alleen-lezen, toont nooit prompts en past nooit veilige migraties toe. Gebruik `doctor --fix` of `doctor --repair` wanneer je wilt dat doctor wijzigingen aanbrengt.
- Doctor voert bij het standaard controleren van geheimen geen `exec` SecretRefs uit. Gebruik `--allow-exec` (met of zonder `--lint`) alleen wanneer je doctor bewust die geconfigureerde geheimoplossers wilt laten uitvoeren.
- Bij elke configuratieschrijfactie (inclusief een reparatie met `--fix`) wordt een back-up geroteerd naar `~/.openclaw/openclaw.json.bak` (met een genummerde ring van `.bak.1`..`.bak.4`). `--fix` verwijdert ook onbekende configuratiesleutels die door schemavalidatie zijn gemeld en vermeldt elke verwijdering; dit wordt overgeslagen terwijl een update wordt uitgevoerd, zodat gedeeltelijk geschreven upgradestatus niet wordt verwijderd voordat de migratie is voltooid.
- Als `openclaw.json` niet kan worden geparseerd en geen laatst bekende werkende configuratie kan worden hersteld, bewaart `doctor --fix` het origineel als `openclaw.json.clobbered.<timestamp>`, laat het huidige bestand ongewijzigd en sluit af met een fout in plaats van een gedeeltelijke vervanging te schrijven.
- Stel `OPENCLAW_SERVICE_REPAIR_POLICY=external` in wanneer een andere supervisor de levenscyclus van de Gateway beheert. Doctor rapporteert nog steeds de status van Gateway/services en past reparaties buiten services toe, maar slaat installatie/start/herstart/bootstrap van services en het opschonen van verouderde services over.
- Doctor rapporteert de toegepaste heaplimiet van de beheerde Gateway en de adaptieve afleiding die voor de huidige geheugenlimiet van de host of container wordt gebruikt. Gebruik `openclaw gateway status` voor hetzelfde rapport buiten een reparatieronde.
- Op Linux negeert doctor inactieve extra systemd-eenheden die op een Gateway lijken en herschrijft tijdens reparatie geen opdracht-/toegangspuntmetadata voor een actieve systemd-Gatewayservice. Stop eerst de service of gebruik `openclaw gateway install --force` om het actieve startprogramma te vervangen.
- `doctor --fix --non-interactive` rapporteert ontbrekende of verouderde Gateway-servicedefinities, maar installeert of herschrijft ze niet buiten de reparatiemodus voor updates. Voer `openclaw gateway install` uit voor een ontbrekende service of `openclaw gateway install --force` om het startprogramma te vervangen.
- Controles van de statusintegriteit detecteren verweesde transcriptbestanden in de sessiemap. Voor archivering ervan als `.deleted.<timestamp>` is interactieve bevestiging vereist; `--fix`, `--yes` en headless-uitvoeringen laten ze staan.
- Doctor scant `~/.openclaw/cron/jobs.json` (of `cron.store`) op verouderde cron-taakstructuren en herschrijft deze voordat canonieke rijen in SQLite worden geïmporteerd.
- Doctor rapporteert cron-taken met een expliciete `payload.model`-overschrijving, inclusief aantallen per providernamespace en afwijkingen ten opzichte van `agents.defaults.model`, zodat geplande taken die het standaardmodel niet overnemen zichtbaar zijn tijdens onderzoek naar authenticatie of facturering.
- Doctor rapporteert cron-taken die nog als actief zijn gemarkeerd (`state.runningAtMs`), waardoor `openclaw cron list` ze als `running` kan weergeven. Deze controle is alleen-lezen: als momenteel geen Gateway een gemarkeerde taak uitvoert, registreert de volgende start van de cron-service de onderbroken uitvoering en wist deze de markering.
- Op Linux waarschuwt doctor wanneer de crontab van de gebruiker nog steeds de niet-onderhouden verouderde `~/.openclaw/bin/ensure-whatsapp.sh` uitvoert, die `Gateway inactive` onjuist kan rapporteren wanneer cron niet over de systemd-gebruikersbusomgeving beschikt.
- Wanneer WhatsApp is ingeschakeld, controleert doctor op een verslechterde Gateway-gebeurtenislus terwijl lokale `openclaw-tui`-clients nog actief zijn. `doctor --fix` stopt alleen geverifieerde lokale TUI-clients, zodat WhatsApp-antwoorden niet achter verouderde TUI-verversingslussen in de wachtrij komen.
- Wanneer HTTP(S)-proxyomgevingsvariabelen aanwezig zijn maar `tools.web.fetch.useTrustedEnvProxy` is uitgeschakeld, legt doctor uit dat `web_fetch` nog steeds directe routering gebruikt, voert het een korte directe TLS-verbindingscontrole uit en noemt het de expliciete opt-in. Proxyvertrouwen wordt nooit automatisch ingeschakeld.
- Doctor herschrijft verouderde `codex/*`- en `openai-codex/*`-modelverwijzingen naar canonieke `openai/*`-verwijzingen voor primaire modellen, fallbacks, modeltoelatingslijsten, modellen voor beeld-/videogeneratie, Heartbeat-/subagent-/Compaction-overschrijvingen, hooks, kanaalmodeloverschrijvingen, cron-payloads en verouderde routepinnen voor sessies/transcripten. `--fix` voegt ook veilig verouderde configuratie van `models.providers.codex` en `models.providers.openai-codex` samen, migreert verouderde authenticatieprofielen van `openai-codex:*` en `auth.order.openai-codex`-vermeldingen naar `openai:*`, verplaatst Codex-intentie naar provider-/modelgebonden `agentRuntime.id: "codex"`-vermeldingen, verwijdert verouderde runtimepinnen voor volledige agents/sessies en houdt gerepareerde OpenAI-agentverwijzingen op Codex-authenticatieroutering in plaats van directe authenticatie met een OpenAI-API-sleutel.
- Doctor rapporteert niet-lege `auth.order.<provider>`-lijsten waarvan alle profielen waarnaar wordt verwezen verdwenen zijn, terwijl compatibele opgeslagen inloggegevens bestaan. `doctor --fix` verwijdert alleen die verouderde overschrijvingen en herstelt daarmee de automatische selectie van inloggegevens per agent; expliciete lege volgorden, gedeeltelijk actieve lijsten en volgorden zonder compatibele opgeslagen inloggegevens blijven ongewijzigd. Als een actieve SQLite-authenticatieopslag onleesbaar of onjuist gevormd is, legt doctor uit waarom deze reparatie is overgeslagen. Herstart een actieve Gateway voordat je de authenticatiestatus opnieuw controleert als de herlaadmodus voor de configuratie de schrijfactie niet automatisch toepast.
- Doctor ruimt verouderde voorbereidingsstatus van plugin-afhankelijkheden uit oudere OpenClaw-versies op en koppelt het `openclaw`-pakket van de host opnieuw voor beheerde npm-plugins die dit als peerafhankelijkheid declareren. Het repareert ook ontbrekende downloadbare plugins waarnaar de configuratie verwijst (`plugins.entries`, geconfigureerde kanalen, geconfigureerde provider-/zoekinstellingen en geconfigureerde agentruntimes). Tijdens pakketupdates slaat doctor reparatie van plugins via de pakketbeheerder over totdat de pakketwisseling is voltooid; voer daarna `openclaw doctor --fix` opnieuw uit als een geconfigureerde plugin nog moet worden hersteld. Als een download mislukt, rapporteert doctor de installatiefout en behoudt het de geconfigureerde pluginvermelding voor de volgende reparatiepoging.
- Doctor repareert verouderde pluginconfiguratie door ontbrekende plugin-id's te verwijderen uit `plugins.allow`/`plugins.deny`/`plugins.entries`, samen met overeenkomende loshangende kanaalconfiguratie, Heartbeat-doelen en kanaalmodeloverschrijvingen, wanneer plugindetectie correct werkt.
- Doctor plaatst ongeldige pluginconfiguratie in quarantaine door de betreffende `plugins.entries.<id>`-vermelding uit te schakelen en de ongeldige `config`-payload ervan te verwijderen. Bij het starten slaat de Gateway alleen die onjuiste plugin al over, zodat andere plugins en kanalen actief blijven.
- Doctor verwijdert de buiten gebruik gestelde `plugins.entries.codex.config.codexDynamicToolsProfile`; de Codex-app-server houdt werkruimtetools die eigen zijn aan Codex altijd native.
- Doctor migreert verouderde platte Talk-configuratie (`talk.voiceId`, `talk.modelId` en verwante instellingen) automatisch naar `talk.provider` + `talk.providers.<provider>`. Herhaalde uitvoeringen van `doctor --fix` rapporteren/passen geen Talk-normalisatie meer toe wanneer alleen de sleutelvolgorde van objecten verschilt.
- Doctor bevat een gereedheidscontrole voor geheugenzoekopdrachten en kan `openclaw configure --section model` aanbevelen wanneer inloggegevens voor embeddings ontbreken.
- Doctor waarschuwt wanneer geen opdrachteigenaar is geconfigureerd. De opdrachteigenaar is het account van de menselijke beheerder dat alleen-voor-eigenaar-opdrachten mag uitvoeren en gevaarlijke acties mag goedkeuren. DM-koppeling staat alleen toe dat iemand met de bot praat; als je een afzender hebt goedgekeurd voordat bootstrap voor de eerste eigenaar bestond, stel dan `commands.ownerAllowFrom` expliciet in.
- Doctor rapporteert een informatieve melding wanneer agents in Codex-modus zijn geconfigureerd en persoonlijke Codex CLI-assets in de Codex-basismap van de beheerder aanwezig zijn. Lokale Codex-app-serverstarts gebruiken geïsoleerde basismappen per agent; installeer indien nodig eerst de Codex-plugin en gebruik daarna `openclaw migrate plan codex` om assets te inventariseren die bewust moeten worden gepromoveerd.
- Doctor waarschuwt wanneer Skills die voor de standaardagent zijn toegestaan niet beschikbaar zijn in de huidige runtimeomgeving (ontbrekende uitvoerbare bestanden, omgevingsvariabelen, configuratie of OS-vereisten). `doctor --fix` kan die niet-beschikbare Skills uitschakelen met `skills.entries.<skill>.enabled=false`; installeer/configureer in plaats daarvan de ontbrekende vereiste als je de Skill actief wilt houden.
- Als de sandboxmodus is ingeschakeld maar Docker niet beschikbaar is, rapporteert doctor een duidelijke waarschuwing met herstelopties (`install Docker` of `openclaw config set agents.defaults.sandbox.mode off`).
- Als verouderde sandboxregisterbestanden of shardmappen aanwezig zijn (`~/.openclaw/sandbox/containers.json`, `~/.openclaw/sandbox/browsers.json`, `~/.openclaw/sandbox/containers/` of `~/.openclaw/sandbox/browsers/`), rapporteert doctor deze; `--fix` migreert geldige vermeldingen naar SQLite en plaatst ongeldige verouderde bestanden in quarantaine.
- Als `gateway.auth.token`/`gateway.auth.password` door SecretRef worden beheerd en niet beschikbaar zijn in het huidige opdrachtpad, rapporteert doctor een alleen-lezenwaarschuwing en schrijft het geen platte-tekstfallbackinloggegevens. Voor door exec ondersteunde SecretRefs slaat doctor de uitvoering over tenzij `--allow-exec` aanwezig is.
- Als inspectie van een kanaal-SecretRef in een reparatiepad mislukt, gaat doctor door en rapporteert het een waarschuwing in plaats van vroegtijdig af te sluiten.
- Na migraties van de statusmap waarschuwt doctor wanneer ingeschakelde standaardaccounts van Telegram of Discord afhankelijk zijn van een omgevingsfallback en `TELEGRAM_BOT_TOKEN` of `DISCORD_BOT_TOKEN` niet beschikbaar is voor het doctor-proces.
- Voor automatische omzetting van Telegram-`allowFrom`-gebruikersnamen (`doctor --fix`) is een omzetbaar Telegram-token in het huidige opdrachtpad vereist. Als tokeninspectie niet beschikbaar is, rapporteert doctor een waarschuwing en slaat het de automatische omzetting voor die ronde over.

## macOS: omgevingsoverschrijvingen voor `launchctl`

Als je eerder `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...` (of `...PASSWORD`) hebt uitgevoerd, overschrijft die waarde je configuratiebestand en kan dit aanhoudende fouten met "niet geautoriseerd" veroorzaken.

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Gateway doctor](/nl/gateway/doctor)
