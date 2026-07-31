---
read_when:
    - Achtergrondtaken of wake-ups plannen
    - Externe triggers (webhooks, Gmail) koppelen aan OpenClaw
    - Kiezen tussen Heartbeat en Cron voor geplande taken
sidebarTitle: Scheduled tasks
summary: Geplande taken, webhooks en Gmail PubSub-triggers voor de Gateway-planner
title: Geplande taken
x-i18n:
    generated_at: "2026-07-27T06:03:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd889cf8e45196eda3ec7c2af930abcb2cc2bae8bad2dbdcaf3cd521a9e884b2
    source_path: automation/cron-jobs.md
    workflow: 16
---

Cron is de ingebouwde planner van de Gateway. Cron bewaart taken, wekt de agent op het juiste moment en kan uitvoer afleveren bij een chatkanaal, een Webhook of nergens.

## Snel aan de slag

<Steps>
  <Step title="Een eenmalige herinnering toevoegen">
    ```bash
    openclaw cron create "2027-02-01T16:00:00Z" \
      --name "Reminder" \
      --session main \
      --system-event "Reminder: check the cron docs draft" \
      --wake now \
      --delete-after-run
    ```
  </Step>
  <Step title="Je taken controleren">
    ```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  </Step>
  <Step title="Uitvoeringsgeschiedenis bekijken">
    ```bash
    openclaw cron runs --id <job-id>
    ```
  </Step>
</Steps>

## Hoe Cron werkt

- Cron wordt **binnen het Gateway-proces** uitgevoerd, niet binnen het model. De Gateway moet actief zijn om planningen te activeren.
- Taakdefinities, runtimestatus en uitvoeringsgeschiedenis blijven bewaard in de gedeelde SQLite-statusdatabase van OpenClaw, zodat planningen bij herstarts niet verloren gaan.
- Elke Cron-uitvoering maakt een [achtergrondtaak](/nl/automation/tasks)-record.
- Eenmalige taken (`--at`) worden na succes standaard automatisch verwijderd; geef `--keep-after-run` door om ze te behouden.
- Tijdslimiet per uitvoering: `--timeout-seconds` indien ingesteld. Anders worden geïsoleerde/losgekoppelde agentbeurttaken begrensd door Crons eigen waakhond van 60 minuten, voordat de onderliggende time-out voor de agentbeurt (`agents.defaults.timeoutSeconds`, standaard 48 uur) ooit van toepassing zou zijn; opdrachttaken hebben standaard een limiet van 10 minuten en scriptpayloads van 5 minuten.
- Bij het starten van de Gateway worden achterstallige geïsoleerde agentbeurttaken opnieuw gepland in plaats van onmiddellijk opnieuw afgespeeld, zodat het initialisatiewerk voor modellen en tools buiten het venster voor kanaalverbindingen blijft.
- Als je `openclaw agent` vanuit systeem-Cron of een andere externe planner aanstuurt, omhul dit dan met een escalatie die het proces geforceerd beëindigt, ook al verwerkt de CLI `SIGTERM`/`SIGINT` al. Door de Gateway ondersteunde uitvoeringen vragen de Gateway om geaccepteerde uitvoeringen af te breken; `--local`-uitvoeringen krijgen hetzelfde afbreeksignaal. Geef voor GNU `timeout` de voorkeur aan `timeout -k 60 600 openclaw agent ...` boven alleen `timeout 600 ...` — de waarde `-k` is het vangnet als het proces niet tijdig kan worden afgerond. Gebruik voor systemd-eenheden een `SIGTERM`-stopsignaal met een respijtperiode (`TimeoutStopSec`) vóór de definitieve beëindiging. Als je een `--run-id` hergebruikt terwijl de oorspronkelijke Gateway-uitvoering nog actief is, wordt het duplicaat als actief gemeld in plaats van dat een tweede uitvoering wordt gestart.

<AccordionGroup>
  <Accordion title="Versterking van geïsoleerde uitvoeringen">
    - Geïsoleerde uitvoeringen proberen bij voltooiing zo goed mogelijk bijgehouden browsertabbladen/-processen voor hun `cron:<jobId>`-sessie te sluiten en ruimen alle voor de taak gemaakte gebundelde MCP-runtime-instanties op via hetzelfde gedeelde afbraakpad dat wordt gebruikt voor uitvoeringen in de hoofdsessie en aangepaste sessies. Opruimfouten worden genegeerd, zodat het Cron-resultaat leidend blijft.
    - Geïsoleerde uitvoeringen met de beperkte toestemming voor zelfopruiming van Cron kunnen de plannerstatus lezen, een op zichzelf gefilterde lijst met alleen hun eigen taak en de uitvoeringsgeschiedenis van die taak, en mogen uitsluitend hun eigen taak verwijderen.
    - Geïsoleerde uitvoeringen beschermen tegen verouderde bevestigingsantwoorden: als het eerste resultaat slechts een tussentijdse statusupdate is (`on it`, `pulling everything together` en vergelijkbare aanwijzingen) en geen onderliggende subagent nog verantwoordelijk is voor het definitieve antwoord, vraagt OpenClaw één keer opnieuw om het daadwerkelijke resultaat voordat dit wordt afgeleverd.
    - Gestructureerde metadata over geweigerde uitvoering (waaronder node-host-`UNAVAILABLE`-wrappers waarvan de geneste fout begint met `SYSTEM_RUN_DENIED` of `INVALID_REQUEST`) wordt herkend, zodat een geblokkeerde opdracht niet als een geslaagde uitvoering wordt gemeld, terwijl gewone assistentproza niet ten onrechte als een weigering wordt beschouwd.
    - Agentfouten op uitvoeringsniveau tellen als taakfouten, zelfs zonder antwoordpayload, zodat model-/providerfouten de fouttellers verhogen en foutmeldingen activeren in plaats van de taak als geslaagd af te handelen.
    - Wanneer een taak `timeoutSeconds` bereikt, breekt Cron de uitvoering af en geeft deze een kort opruimvenster. Als de uitvoering niet wordt afgerond, wist door de Gateway beheerde opruiming geforceerd het sessie-eigenaarschap van die uitvoering voordat Cron de time-out registreert, zodat chatwerk in de wachtrij niet vastloopt achter een verouderde verwerkende sessie.
    - Vastlopers tijdens configuratie/opstart krijgen een fasespecifieke time-out (bijvoorbeeld `cron: isolated agent setup timed out before runner start` of `cron: isolated agent run stalled before execution start (last phase: context-engine)`). Deze waakhonden dekken ingebedde en door de CLI ondersteunde providers, zelfs voordat hun externe CLI-proces start, en worden onafhankelijk van lange `timeoutSeconds`-waarden begrensd, zodat fouten bij een koude start, authenticatie of context snel zichtbaar worden.

  </Accordion>
  <Accordion title="Taakreconciliatie">
    Reconciliatie van Cron-taken wordt primair door de runtime beheerd en secundair ondersteund door duurzame geschiedenis: een actieve Cron-taak blijft actief zolang de Cron-runtime die taak nog als actief bijhoudt, zelfs als er nog een oude rij voor een onderliggende sessie bestaat. Zodra de runtime niet langer eigenaar is van de taak en een respijtperiode van 5 minuten is verstreken, controleert onderhoud de bewaarde uitvoeringslogboeken en taakstatus voor de overeenkomende `cron:<jobId>:<startedAt>`-uitvoering. Een eindresultaat daarin voltooit het taakregister; anders kan door de Gateway beheerd onderhoud de taak markeren als `lost`. Een offline CLI-audit kan herstel uitvoeren op basis van duurzame geschiedenis, maar de eigen lege verzameling actieve taken in het proces bewijst niet dat een door de Gateway beheerde uitvoering verdwenen is.
  </Accordion>
</AccordionGroup>

## Planningstypen

| Soort      | CLI-vlag           | Beschrijving                                                                                              |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| `at`      | `--at`             | Eenmalig tijdstip (ISO 8601 of relatief, zoals `20m`)                                                     |
| `every`   | `--every`          | Vast interval (`10m`, `1h`, `1d`)                                                                       |
| `cron`    | `--cron`           | Cron-expressie met 5 of 6 velden en optionele `--tz`                                                  |
| `on-exit` | `--on-exit`        | Eenmalig activeren wanneer een bewaakte opdracht eindigt (gebeurtenistrigger; blijft bestaan na afbraak van de beurt; optionele `--on-exit-cwd`) |
| `stream`  | `--stream-command` | Activeren vanuit gebundelde regels die door een bewaakte langlopende opdracht worden geproduceerd                                      |

Tijdstippen zonder tijdzone worden als UTC behandeld. Voeg `--tz America/New_York` toe om een `--at`-datum/tijd zonder offset in die IANA-tijdzone te interpreteren, of om daarin een Cron-expressie te evalueren. Cron-expressies zonder `--tz` gebruiken de tijdzone van de Gateway-host. `--tz` is niet geldig met `--every` of `--on-exit`.

Terugkerende expressies op het hele uur (minuut `0` met een jokerteken in het uurveld) worden automatisch over maximaal 5 minuten gespreid om belastingpieken te verminderen. Gebruik `--exact` om exacte timing af te dwingen, of `--stagger 30s` voor een expliciet venster (alleen Cron-planningen).

### Migratie van Heartbeat-taken

Oudere Heartbeat-kladgegevens ondersteunden een gestructureerd `tasks:`-blok. Voer na de upgrade `openclaw doctor --fix` uit om elk item om te zetten in een gewone, bewerkbare Cron-taak voor de hoofdsessie. Doctor behoudt het interval en het vorige tijdstip van de laatste uitvoering, maakt de taken voordat het blok wordt verwijderd en brengt dezelfde declaratiesleutels bij een nieuwe uitvoering veilig naar dezelfde toestand.

Deze gemigreerde taken bevatten openbare `systemEvent`-payloads, zodat `openclaw cron list`, `get`, `edit` en `remove` plus de Cron-tool ze net als andere taken beheren. Voor hun uitvoering wordt de beveiligde wekactie voor Heartbeat-taken gebruikt: actieve uren, minimale tussenruimte, overstromingsbeheer en nieuwe pogingen bij drukte blijven van toepassing, terwijl Cron het onafhankelijke ritme van elke taak beheert. Taken die in hetzelfde samenvoegvenster vervallen, kunnen één Heartbeat-beurt delen. Een geplande uitvoering buiten de actieve uren van Heartbeat wordt overgeslagen en opnieuw geprobeerd bij de volgende uitvoering van de taak.

Heartbeat-kladgegevens zijn nu uitsluitend tekst voor bewaking. Runtime-Heartbeats ontleden `tasks:`-tekst niet als planningen; maak nieuw terugkerend werk met Cron.

### Streambronnen

Een streamplanning houdt een door een beheerder opgestelde argv-opdracht actief onder de Gateway en activeert de taak op basis van regels uit stdout en stderr. Streamplanningen zijn gebeurtenisgestuurd, nooit tijdsgebonden en vereisen `cron.triggers.enabled: true`, omdat de langlopende opdracht dezelfde vertrouwensklasse voor onbeheerde uitvoering heeft als triggerscripts. Als je de taak uitschakelt of verwijdert, wordt het proces gestopt; bij het afsluiten van de Gateway wordt gewacht tot de procesboom is afgebroken. Snelle fouten worden opnieuw gestart met Crons ingebouwde foutvertraging. Vijf opeenvolgende uitvoeringen die korter dan 60 seconden duren, laten de taak in een foutstatus achter en gebruiken het normale waarschuwingspad voor fouten; schakel de taak handmatig opnieuw in om de herstartlimiet te wissen.

```bash
openclaw cron add \
  --name "Build event stream" \
  --stream-command '["node","scripts/build-events.mjs"]' \
  --stream-mode match \
  --stream-match '^(failed|recovered):' \
  --stream-batch-ms 250 \
  --session isolated \
  --message "Investigate these build events."
```

`mode: "line"` (de standaardinstelling) accepteert elke regel. `mode: "match"` accepteert alleen regels die overeenkomen met de gecompileerde `match`-regex. Een batch wordt gesloten na `batchMs` stilte (standaard 250 ms, begrensd op 50–5000) of bij `maxBatchBytes` (standaard 16384, begrensd op 1024–65536). Bij de bytegrens eindigt de batch met `[truncated]`. De overeenkommodus evalueert altijd volledige regels aan de hand van hun volledige tekst, zelfs voorbij `maxBatchBytes` (alleen de afgeleverde batch wordt afgekapt); een regel die bij de begrensde limiet voor ruwe invoer wordt afgekapt, is slechts een voorvoegsel en wordt daarom als niet-overeenkomend behandeld, zodat een aan het einde verankerd patroon niet op de afkapping kan worden geactiveerd. De batch wordt toegevoegd aan de tekst van de systeemgebeurtenis of het agentbeurtbericht. Opdrachtpayloads worden voor streamplanningen geweigerd, omdat het broncommando en het payloadcommando anders dubbelzinnig proceseigenaarschap zouden hebben.

Per taak worden slechts één payloadactivering en één begrensde wachtende batch bewaard. Regels die binnenkomen terwijl een payload wordt uitgevoerd, of voordat het ingebouwde triggerinterval van 30 seconden is verstreken, worden samengevoegd in die wachtende batch in plaats van een onbegrensde wachtrij op te bouwen. Eén geserialiseerde eigenaar registreert weigeringen door de poort, payloadfouten en verzendingen terwijl de taak niet actief is in `streamDroppedBatches`; begrensde samenvoegingen verhogen `streamCoalescedBatches`. Mislukte payloads worden niet opnieuw geprobeerd, omdat ze mogelijk niet idempotent zijn. Een logische bronidentiteit blijft stabiel tijdens herstarts van bewaakte onderliggende processen, maar wordt vervangen wanneer de bron wordt uitgeschakeld, verwijderd of vervangen, zodat batches in de wachtrij van de buiten gebruik gestelde bron niet kunnen worden geactiveerd, zelfs niet na een bewerking van A naar B naar A. Nadat het stoppen is voltooid, hebben late callbacks van een oud onderliggend proces geen effect. V1 bevat geen ingebouwde WebSocket-bron; overbrug er een met een argv-opdracht zoals `websocat wss://example.invalid/events`.

Wanneer een streamtaak ook `trigger.script` heeft, wordt de poort eenmaal per gesloten batch uitgevoerd. De huidige batch is beschikbaar als de diep bevroren `trigger.streamBatch`-tekenreeks naast `trigger.state`. `fire: false` verwijdert die batch nadat de poortstatus is bewaard. `fire: true` behoudt de bestaande semantiek van triggerberichten en voegt vervolgens de batch toe aan de resulterende payload. Een streamtaak kan in plaats daarvan een scriptpayload zonder voorwaardepoort gebruiken; dat script ontvangt de batch via dezelfde `trigger.streamBatch`-waarde. Het combineren van een scriptpayload met een voorwaardepoort wordt geweigerd, omdat beide eigenaar zouden zijn van het bewaarde `trigger.state`-slot.

### Dynamisch ritme (dosering)

Terugkerende taken kunnen `pacing.min` en/of `pacing.max` instellen op duurtekenreeksen zoals `15m` of `4h`; er is ten minste één grens vereist. Gebruik `--pacing-min` en `--pacing-max` met `cron add|edit` (`--clear-pacing` verwijdert beide grenzen).

Tijdens een geïsoleerde uitvoering kan een getemporiseerde taak de tool `cron` aanroepen met `action: "next_check"` en `in: "30m"`. Het voorstel is alleen van toepassing op die momenteel uitgevoerde taak en wordt gemeten vanaf de succesvolle voltooiing van de uitvoering. OpenClaw begrenst dit stilzwijgend tot de geconfigureerde limieten.

Temporisering zonder voorstel laat het normale schema ongewijzigd. Mislukte uitvoeringen, uitvoeringen met een time-out en overgeslagen uitvoeringen verwerpen het voorstel, zodat bestaand gedrag voor nieuwe pogingen en foutgerelateerde back-off voorrang krijgt. Het handmatig afdwingen van een terugkerende taak valt buiten de normale planning en behoudt het wachtende natuurlijke of getemporiseerde tijdslot. Voor voorwaardelijk geactiveerde taken blijft het ingebouwde minimuminterval een ondergrens, zelfs wanneer een voorstel om een eerdere controle vraagt.

### Dag van de maand en dag van de week gebruiken OF-logica

Cron-expressies worden geparseerd door [croner](https://github.com/Hexagon/croner). Wanneer zowel het veld voor de dag van de maand als dat voor de dag van de week geen jokerteken bevat, is er volgens croner een overeenkomst wanneer **een van beide** velden overeenkomt, niet wanneer beide overeenkomen. Dit is standaardgedrag van Vixie cron.

```bash
# Bedoeld: "9.00 uur op de 15e, alleen als het een maandag is"
# Werkelijk: "9.00 uur op elke 15e, EN 9.00 uur op elke maandag"
0 9 15 * 1
```

Dit wordt ongeveer 5-6 keer per maand geactiveerd in plaats van 0-1 keer per maand. Gebruik croners dag-van-de-weekmodifier `+` (`0 9 15 * +1`) om beide voorwaarden te vereisen, of plan op basis van het ene veld en controleer het andere in de prompt of opdracht van je taak.

## Gebeurtenistriggers (conditiebewakers)

Een gebeurtenistrigger voegt een headless conditiescript toe aan een schema van het type `every`, `cron` of `stream`. Tijdschema's evalueren het wanneer het tijdstip is aangebroken; streamschema's evalueren het voor elke afgesloten batch. Cron voert de normale payload alleen uit wanneer het script `fire: true` retourneert:

```json5
{
  schedule: { kind: "every", everyMs: 30000 },
  trigger: {
    // Wordt alleen geactiveerd wanneer de waargenomen status afwijkt van de vorige evaluatie.
    script: "const res = await tools.call('exec', { command: 'gh pr checks 123 --json state -q \\'.[].state\\' | sort -u' }); const status = String(res?.result?.details?.aggregated ?? '').trim(); json({ fire: status !== trigger.state?.status, message: `PR 123 CI: ${trigger.state?.status ?? 'unknown'} -> ${status}`, state: { status } });",
    once: false,
  },
  payload: { kind: "agentTurn", message: "Onderzoek de wijziging in de CI-status." },
}
```

Het script moet `{ fire, message?, state? }` retourneren. De vorige JSON-status is beschikbaar als de diep bevroren `trigger.state`; streamgates ontvangen ook de huidige batch als `trigger.streamBatch`. Retourneer een nieuwe waarde voor `state` om deze persistent op te slaan. De status is beperkt tot 16 KB. Wanneer een activeringsresultaat `message` bevat, voegt cron dit vóór de uitvoering toe aan de tekst van de systeemgebeurtenis of het agentbericht. `once: true` schakelt de taak uit na de eerste succesvol geactiveerde payload.

`fire: false` slaat de evaluatiestatus en tellers persistent op en plant vervolgens opnieuw zonder uitvoeringsgeschiedenis aan te maken. Als de uitvoering van een geactiveerde payload mislukt, wordt de geretourneerde `state` **niet** persistent opgeslagen — de volgende evaluatie ziet de vorige status en kan opnieuw worden geactiveerd. Schrijf scripts daarom als alleen-lezencontroles en houd acties in de payload. Triggerschema's hebben een ingebouwd minimuminterval van 30 seconden. Elke evaluatie heeft een wandklokbudget van 30 seconden en maximaal 5 toolaanroepen.

Ontwerp bewakers rond **actiegerichte status**, niet alleen rond succes: een bewaker die stilvalt wanneer de controle mislukt of een time-out bereikt, lijkt gezond terwijl hij defect is. Vergelijk de waarneming met `trigger.state` en retourneer een nieuwe status om duplicaten te voorkomen; vertrouw niet op het geheugen van het model of proces. Maak bij activering `message` zelfstandig begrijpelijk, omdat dit de volledige gebeurteniscontext van de geactiveerde uitvoering wordt.

<Warning>
Als je `cron.triggers.enabled` inschakelt, mogen zowel scripts voor voorwaardelijke triggers als `script`-payloads headless worden uitgevoerd met het **volledige toolbeleid van de eigenaaragent, inclusief `exec`**. Beschouw dit als uitvoering van code zonder toezicht met de machtigingen van die agent; laat dit uitgeschakeld tenzij elke agent die cron-taken mag maken dienovereenkomstig wordt vertrouwd.
</Warning>

Maak een bewaker vanuit een lokaal scriptbestand (`-` leest het script uit stdin):

```bash
openclaw cron add \
  --name "PR CI-bewaker" \
  --every 30s \
  --trigger-script ./watch-pr-ci.js \
  --message "Reageer op de wijziging in de CI-status" \
  --session isolated
```

## Payloads

Elke taak bevat precies één payloadtype, gekozen via een vlag:

| Payload       | Vlag                                           | Wordt uitgevoerd als                                        |
| ------------- | ---------------------------------------------- | ----------------------------------------------------------- |
| Systeemgebeurtenis | `--system-event <text>`                        | In de hoofdsessie in de wachtrij geplaatst, zonder zelfstandige modelaanroep |
| Agentbericht | `--message <text>`                             | Een door een model ondersteunde agentbeurt                   |
| Opdracht       | `--command <shell>` of `--command-argv <json>` | Een shell/proces op de Gateway-host, zonder modelaanroep     |
| Script        | `--script <file\|->`                           | Een headless code-modusscript dat de tools van de eigenaaragent gebruikt |

Een aanvullend payloadtype, `heartbeat`, is systeemeigendom: de Gateway convergeert naar één Heartbeat-bewakingstaak per agent waarvoor Heartbeat is ingeschakeld (zie [Heartbeat](/nl/gateway/heartbeat)). Deze verschijnt in `cron list --all`, maar kan niet via de CLI of API worden gemaakt of bewerkt. De Heartbeat-configuratie wordt bij het opstarten, bij het opnieuw laden van de configuratie of door `openclaw doctor --fix` doorgeschreven naar het persistent opgeslagen bewakingsschema. Wanneer cron is uitgeschakeld, tikt de bewaker niet en wordt er geen alternatieve Heartbeat-timer uitgevoerd.

### Opties voor agentbeurten

<ParamField path="--message" type="string" required>
  Prompttekst (vereist voor geïsoleerde taken, taken in de huidige sessie en taken in aangepaste sessies).
</ParamField>
<ParamField path="--model" type="string">
  Modeloverride; moet worden herleid tot een toegestaan model, anders mislukt de uitvoering met een validatiefout.
</ParamField>
<ParamField path="--fallbacks" type="string">
  Lijst met fallbackmodellen per taak, bijvoorbeeld `--fallbacks openai/gpt-5.6-sol,openrouter/meta-llama/llama-3.3-70b-instruct:free`. Geef `--fallbacks ""` door voor een strikte uitvoering zonder fallbacks.
</ParamField>
<ParamField path="--clear-fallbacks" type="boolean">
  Verwijdert bij `cron edit` de fallbackoverride per taak, zodat de taak de geconfigureerde fallbackprioriteit volgt. Kan niet worden gecombineerd met `--fallbacks`.
</ParamField>
<ParamField path="--clear-model" type="boolean">
  Verwijdert bij `cron edit` de modeloverride per taak, zodat de taak de normale prioriteit voor cronmodellen volgt (opgeslagen cron-sessieoverride, anders agent-/standaardmodel). Kan niet worden gecombineerd met `--model`.
</ParamField>
<ParamField path="--thinking" type="string">
  Override voor het denkniveau (`off|minimal|low|medium|high|xhigh|adaptive|max|ultra`). De beschikbare niveaus zijn nog steeds afhankelijk van het geselecteerde model en de agentruntime.
</ParamField>
<ParamField path="--clear-thinking" type="boolean">
  Verwijdert bij `cron edit` de denkoverride per taak. Kan niet worden gecombineerd met `--thinking`.
</ParamField>
<ParamField path="--light-context" type="boolean">
  Sla het injecteren van bootstrapbestanden voor de werkruimte over.
</ParamField>
<ParamField path="--tools" type="string">
  Beperk welke tools de taak kan gebruiken, bijvoorbeeld `--tools exec,read`.
</ParamField>

Nieuwe taken die tools kunnen uitvoeren, slaan altijd een expliciet toolbeleid op. Taken die door een agent
worden gemaakt, zijn beperkt tot de tools die beschikbaar zijn voor de aanmakende beurt, en de agent kan de
opgeslagen lijst niet uitbreiden. Taken die door een geverifieerde operator zonder `--tools` worden gemaakt, slaan een
onbeperkt `*`-beleid op; `cron edit --clear-tools` herstelt dat expliciete onbeperkte
beleid. Bestaande taken van vóór de invoering van een expliciet toolbeleid behouden hun huidige gedrag
totdat hun toolbeleid expliciet wordt bewerkt of de taak opnieuw wordt gemaakt.

`--model` stelt het primaire model van de taak in; het vervangt geen `/model`-override van een sessie, zodat geconfigureerde fallbackketens er nog steeds bovenop worden toegepast. Een niet-herleidbaar of niet-toegestaan model laat de uitvoering mislukken met een expliciete validatiefout in plaats van stilzwijgend terug te vallen op de standaardwaarde. Als een taak `--model` heeft maar geen expliciete of geconfigureerde fallbacklijst, geeft OpenClaw een lege fallbackoverride door in plaats van stilzwijgend het primaire model van de agent toe te voegen als verborgen doel voor een nieuwe poging.

Prioriteit voor modelselectie bij geïsoleerde taken, van hoog naar laag:

1. Payload `model` per taak (expliciete configuratie; een niet-toegestaan model laat de uitvoering mislukken)
2. Modeloverride van de Gmail-hook (alleen wanneer de uitvoering afkomstig is van Gmail en die override is toegestaan)
3. Door de gebruiker geselecteerde, opgeslagen modeloverride voor de cron-sessie
4. Modelselectie van de agent/standaardselectie

De snelle modus volgt de herleide actieve selectie. Als de configuratie van het geselecteerde model `params.fastMode` bevat, gebruikt geïsoleerde cron dit standaard; een opgeslagen `fastMode`-override van de sessie (en vervolgens een `fastModeDefault` van de agent) heeft in beide richtingen nog steeds voorrang op de modelconfiguratie. De automatische modus gebruikt de `params.fastAutoOnSeconds`-grenswaarde van het model, met standaard 60 seconden.

Als tijdens een uitvoering een actieve overdracht voor een modelwissel plaatsvindt, probeert cron het opnieuw met de gewisselde provider/het gewisselde model en slaat die selectie (en een eventueel nieuw verificatieprofiel) persistent op voor de actieve uitvoering. Nieuwe pogingen zijn begrensd: na de eerste poging plus 2 nieuwe pogingen wegens een wissel breekt cron af in plaats van te blijven herhalen.

Voordat een geïsoleerde uitvoering begint, controleert OpenClaw bereikbare lokale eindpunten voor geconfigureerde `api: "ollama"`- en `api: "openai-completions"`-providers waarvan `baseUrl` loopback, privénetwerk of `.local` is. Deze voorafgaande controle doorloopt de geconfigureerde fallbackketen van de taak en markeert de uitvoering pas als `skipped` wanneer elke kandidaat onbereikbaar is; `--fallbacks ""` beperkt deze doorloop strikt tot alleen het primaire model. Een niet-beschikbaar eindpunt registreert de uitvoering als `skipped` met een duidelijke foutmelding in plaats van een modelaanroep te starten. Het resultaat wordt per eindpunt 5 minuten gecachet (niet per taak of model), zodat veel gelijktijdig geplande taken die dezelfde niet-beschikbare lokale Ollama-/vLLM-/SGLang-/LM Studio-server delen, één probe kosten in plaats van een storm aan verzoeken. Overgeslagen vooraf gecontroleerde uitvoeringen verhogen de back-off voor uitvoeringsfouten niet; stel `failureAlert.includeSkipped` in om herhaalde waarschuwingen over overslaan in te schakelen.

### Opdrachtpayloads

Opdrachtpayloads voeren deterministische scripts uit in de Gateway-planner zonder een door een model ondersteunde beurt te starten. Ze worden uitgevoerd op de Gateway-host, leggen stdout/stderr vast, registreren de uitvoering in de crongeschiedenis en gebruiken dezelfde bezorgmodi `announce`, `webhook` en `none` als taken met agentbeurten.

<Note>
Opdrachtcron is een Gateway-automatiseringsoppervlak voor operatorbeheerders, geen `tools.exec`-aanroep van een agent. Voor het maken, bijwerken, verwijderen of handmatig uitvoeren van cron-taken is `operator.admin` vereist; geplande opdrachtuitvoeringen worden later binnen het Gateway-proces uitgevoerd als die door een beheerder gemaakte automatisering. Het uitvoeringsbeleid voor agents (`tools.exec.mode`, goedkeuringsprompts, toestaanslijsten voor tools per agent) beheert uitvoeringstools die zichtbaar zijn voor het model, niet de opdrachtpayloads van cron.
</Note>

```bash
openclaw cron create "*/15 * * * *" \
  --name "Wachtrijdiepteprobe" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` slaat `argv: ["sh", "-lc", <shell>]` op. Gebruik `--command-argv '["node","scripts/report.mjs"]'` voor exacte argv-uitvoering zonder shellparsing. De optionele `--command-env KEY=VALUE` (herhaalbaar), `--command-input`, `--timeout-seconds` (standaard 10 minuten), `--no-output-timeout-seconds` en `--output-max-bytes` beheren de procesomgeving, stdin en uitvoerlimieten.

De bezorgde tekst wordt afgeleid van de procesuitvoer: niet-lege stdout heeft voorrang; als stdout leeg is en stderr niet leeg is, wordt stderr bezorgd; als beide aanwezig zijn, verzendt cron een klein `stdout:`- / `stderr:`-blok. Afsluitcode `0` registreert de uitvoering als `ok`; een niet-nulafsluitcode, signaal, time-out of time-out wegens ontbrekende uitvoer registreert `error` en kan foutwaarschuwingen activeren. Een opdracht die alleen `NO_REPLY` afdrukt, gebruikt de normale onderdrukking van stille crontokens en plaatst niets terug in de chat.

### Scriptpayloads

Scriptpayloads worden headless uitgevoerd in dezelfde code-mode-executor als triggerscripts, zonder een beurt van een conversatieagent te starten. Schakel `cron.triggers.enabled` in voordat je ze maakt of uitvoert; deze beveiliging voor gevaarlijke automatisering geldt voor zowel triggerscripts als scriptpayloads. Scripttaken ondersteunen alleen de sessiedoelen `main` en `isolated`.

```bash
openclaw cron create "0 * * * *" \
  --name "Hourly queue check" \
  --script ./automation/check-queue.js \
  --script-timeout-seconds 300 \
  --script-tool-budget 50 \
  --session isolated \
  --announce
```

Gebruik `--script <file|->` om JavaScript uit een bestand of stdin te lezen. De time-out is standaard 300 seconden en is begrensd op 900; het toolbudget is standaard 50 aanroepen en is begrensd op 200. Deze payloadbudgetten staan los van de kleinere evaluatiebudgetten van de triggerbeveiliging.

Het script kan een object retourneren met deze optionele velden:

- `notify`: Tekst die wordt afgeleverd via de afleveringsmodus `announce`, `webhook` of `none` van de taak. Als dit wordt weggelaten, wordt niets afgeleverd. Voor een `main`-taak wordt de tekst een systeemgebeurtenis.
- `wake`: `"now"` vraagt om een onmiddellijke Heartbeat nadat `notify` (of een compacte voltooiingsgebeurtenis) in de wachtrij is geplaatst; `"next-heartbeat"` plaatst de gebeurtenis in de wachtrij voor de volgende Heartbeat.
- `state`: JSON-status, begrensd op 16 KB en alleen opgeslagen na een geslaagde uitvoering. De volgende uitvoering ontvangt een bevroren kopie als `trigger.state`, overeenkomstig triggerscripts. Omdat die naamruimte één eigenaar van opgeslagen gegevens heeft, kan een scriptpayload niet worden gecombineerd met een voorwaardetrigger voor dezelfde taak.
- `nextCheck`: Een duur zoals `"15m"`. Dit is alleen geldig voor taken waarvoor pacing is ingeschakeld en gebruikt dezelfde pacingbegrenzing als voorstellen voor agentbeurten.

Exceptions, time-outs, uitgeputte toolbudgetten, ongeldige resultaten en `nextCheck` zonder pacing zijn normale cron-uitvoeringsfouten: ze worden opgenomen in de uitvoeringsgeschiedenis, back-off en afhandeling van foutmeldingen, zonder de geretourneerde status op te slaan.

## Uitvoeringsstijlen

| Stijl            | `--session`-waarde | Wordt uitgevoerd in       | Het meest geschikt voor          |
| ---------------- | ------------------------- | ------------------------- | -------------------------------- |
| Hoofdsessie      | `main`        | Speciale cron-wake-lane   | Herinneringen, systeemgebeurtenissen |
| Geïsoleerd       | `isolated`        | Speciale `cron:<jobId>` | Rapporten, achtergrondtaken      |
| Huidige sessie   | `current`        | Gebonden bij het maken    | Terugkerend contextbewust werk   |
| Aangepaste sessie | `session:custom-id`       | Permanente benoemde sessie | Workflows die voortbouwen op geschiedenis |

<AccordionGroup>
  <Accordion title="Hoofdsessie versus geïsoleerd versus aangepast">
    Taken in de **hoofdsessie** plaatsen een systeemgebeurtenis in een door cron beheerde uitvoeringslane en activeren optioneel de Heartbeat (`--wake now` of `--wake next-heartbeat`). Ze kunnen de laatste afleveringscontext van de doelhoofdsessie gebruiken voor antwoorden, maar voegen routinematige cron-beurten niet toe aan de menselijke chatlane en verlengen de versheid voor dagelijkse/inactiviteitsresets van de doelsessie niet. **Geïsoleerde** taken voeren een speciale agentbeurt uit met een nieuwe sessie. **Aangepaste sessies** (`session:xxx`) behouden context tussen uitvoeringen, waardoor workflows mogelijk zijn zoals dagelijkse stand-ups die voortbouwen op eerdere samenvattingen.

    Cron-gebeurtenissen voor de hoofdsessie zijn op zichzelf staande herinneringen in de vorm van systeemgebeurtenissen. Ze bevatten niet automatisch de standaard Heartbeat-prompt of het kladveld van de Heartbeat-monitor; vermeld dit expliciet in de tekst van de cron-gebeurtenis als een herinnering die context moet raadplegen.

  </Accordion>
  <Accordion title="Wat 'nieuwe sessie' betekent voor geïsoleerde taken">
    Een nieuwe transcript-/sessie-id per uitvoering. OpenClaw neemt veilige voorkeuren over (instellingen voor denken/snel/uitgebreid, labels en expliciet door de gebruiker geselecteerde model-/auth-overschrijvingen), maar neemt geen omgevingscontext van een gesprek over uit een oudere cron-rij: kanaal-/groepsroutering, verzend- of wachtrijbeleid, verhoging, oorsprong of ACP-runtimebinding. Gebruik `current` of `session:<id>` wanneer een terugkerende taak bewust moet voortbouwen op dezelfde gesprekscontext.
  </Accordion>
  <Accordion title="Contract voor onbeheerde uitvoering">
    Geïsoleerde cron- en hook-agentbeurten zijn expliciet onbeheerd: er is niemand aanwezig om om verduidelijking of goedkeuring te vragen. Het uiteindelijke antwoord moet het resultaat zijn en geen plan, bevestiging of verzoek om invoer. De agent retourneert `HEARTBEAT_OK` wanneer niets hoeft te worden gedaan en vermeldt fouten duidelijk; cron beheert het beleid voor nieuwe pogingen en foutmeldingen.

    Voor vertrouwde geplande taken hebben de eigen instructies van de taak voorrang wanneer ze bewust om een vraag of plan vragen, en de agent mag een taak verwijderen die niet langer nodig is. Externe hook-beurten ontvangen alleen het algemene contract voor onbeheerde uitvoering; over de grens voor externe inhoud heen ontvangen ze die overschrijving of richtlijnen voor zelfverwijdering niet.

  </Accordion>
  <Accordion title="Aflevering door subagents en Discord">
    Wanneer geïsoleerde cron-uitvoeringen subagents aansturen, krijgt de uiteindelijke uitvoer van de laatste afstammeling bij aflevering de voorkeur boven verouderde tussentijdse tekst van de bovenliggende agent. Als afstammelingen nog actief zijn, onderdrukt OpenClaw die gedeeltelijke update van de bovenliggende agent in plaats van deze aan te kondigen.

    Voor uitsluitend tekst bevattende Discord-aankondigingsdoelen verzendt OpenClaw de canonieke uiteindelijke assistenttekst één keer, in plaats van zowel gestreamde/tussentijdse tekst als het uiteindelijke antwoord opnieuw af te spelen. Media en gestructureerde Discord-payloads worden nog steeds afzonderlijk afgeleverd, zodat bijlagen en componenten niet verloren gaan.

  </Accordion>
</AccordionGroup>

## Aflevering en uitvoer

| Modus      | Wat er gebeurt                                                       |
| ---------- | -------------------------------------------------------------------- |
| `announce` | Levert de uiteindelijke tekst als fallback af aan het doel als de agent deze niet heeft verzonden |
| `webhook` | Verzendt de payload van de voltooide gebeurtenis via POST naar een URL |
| `none` | Geen fallbackaflevering door de runner                              |

Gebruik `--announce --channel telegram --to "-1001234567890"` voor aflevering via een kanaal. Gebruik voor Telegram-forumonderwerpen `-1001234567890:topic:123`; OpenClaw accepteert ook de door Telegram beheerde verkorte vorm `-1001234567890:123`. Directe RPC-/configuratieaanroepers kunnen `delivery.threadId` als tekenreeks of getal doorgeven. Doelen voor Slack/Discord/Mattermost gebruiken expliciete voorvoegsels (`channel:<id>`, `user:<id>`). Matrix-ruimte-id's zijn hoofdlettergevoelig; gebruik de exacte ruimte-id of de vorm `room:!room:server` van Matrix.

Wanneer de aankondigingsaflevering `channel: "last"` gebruikt of `channel` weglaat, kan een doel met providervoorvoegsel zoals `telegram:123` het kanaal selecteren voordat cron terugvalt op de sessiegeschiedenis of één geconfigureerd kanaal. Alleen voorvoegsels die door de geladen Plugin worden aangeboden, zijn providerselectoren. Als `delivery.channel` expliciet is, moet het doelvoorvoegsel dezelfde provider noemen; `channel: "whatsapp"` met `to: "telegram:123"` wordt geweigerd in plaats van WhatsApp de Telegram-id als telefoonnummer te laten interpreteren. Voorvoegsels voor doeltypen en services (`channel:<id>`, `user:<id>`, `imessage:<handle>`, `sms:<number>`) blijven kanaalspecifieke doelsyntaxis en zijn geen providerselectoren.

Voor geïsoleerde taken wordt chataflevering gedeeld: als er een chatroute beschikbaar is, kan de agent de tool `message` zelfs met `--no-deliver` gebruiken. Als de agent naar het geconfigureerde/huidige doel verzendt, slaat OpenClaw de fallbackaankondiging over. Anders bepalen `announce`, `webhook` en `none` alleen wat de runner na de agentbeurt met het uiteindelijke antwoord doet.

Wanneer een agent vanuit een actieve chat een geïsoleerde herinnering maakt, slaat OpenClaw het behouden live afleveringsdoel op voor de fallbackroute voor aankondigingen. Interne sessiesleutels kunnen kleine letters bevatten; providerafleveringsdoelen worden niet opnieuw samengesteld uit die sleutels wanneer de huidige chatcontext beschikbaar is.

Impliciete aankondigingsaflevering gebruikt geconfigureerde kanaaltoelatingslijsten om verouderde doelen te valideren en opnieuw te routeren. Goedkeuringen uit de DM-koppelingsopslag zijn geen ontvangers voor fallbackautomatisering; stel `delivery.to` in of configureer de kanaalvermelding `allowFrom` wanneer een geplande taak proactief naar een DM moet verzenden.

### Foutmeldingen

Foutmeldingen volgen een afzonderlijk bestemmingspad:

- `cron.failureDestination` stelt een algemene standaard voor foutmeldingen in.
- `job.delivery.failureDestination` overschrijft dit per taak.
- Als geen van beide is ingesteld en de taak al via `announce` aflevert, vallen foutmeldingen terug op dat primaire aankondigingsdoel.
- `delivery.failureDestination` wordt alleen ondersteund voor `sessionTarget="isolated"`-taken, tenzij de primaire afleveringsmodus `webhook` is.
- `failureAlert.includeSkipped: true` laat een taak of algemeen cron-waarschuwingsbeleid kiezen voor herhaalde waarschuwingen over overgeslagen uitvoeringen. Overgeslagen uitvoeringen houden een afzonderlijke teller voor opeenvolgende overgeslagen uitvoeringen bij, zodat ze geen invloed hebben op de back-off voor uitvoeringsfouten.
- `openclaw cron edit` maakt waarschuwingafstemming per taak beschikbaar: `--failure-alert`/`--no-failure-alert`, `--failure-alert-after <n>`, `--failure-alert-channel`, `--failure-alert-to`, `--failure-alert-cooldown`, `--failure-alert-include-skipped`/`--failure-alert-exclude-skipped`, `--failure-alert-mode` en `--failure-alert-account-id`.

### Uitvoertaal

Cron-taken leiden geen antwoordtaal af uit het kanaal, de landinstelling of eerdere berichten. Neem de taalregel op in het geplande bericht of de sjabloon:

```bash
openclaw cron edit <jobId> \
  --message "Vat de updates samen. Antwoord in het Chinees; laat URL's, code en productnamen ongewijzigd."
```

Houd voor sjabloonbestanden de taalinstructie in de gerenderde prompt en controleer voordat de taak wordt uitgevoerd of tijdelijke aanduidingen zoals `{{language}}` zijn ingevuld. Als de uitvoer talen mengt, maak de regel dan expliciet, bijvoorbeeld: "Gebruik Chinees voor beschrijvende tekst en behoud technische termen in het Engels."

## CLI-voorbeelden

<Tabs>
  <Tab title="Eenmalige herinnering">
    ```bash
    openclaw cron add \
      --name "Calendar check" \
      --at "20m" \
      --session main \
      --system-event "Next heartbeat: check calendar." \
      --wake now
    ```
  </Tab>
  <Tab title="Terugkerende geïsoleerde taak">
    ```bash
    openclaw cron create "0 7 * * *" \
      "Summarize overnight updates." \
      --name "Morning brief" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --announce \
      --channel slack \
      --to "channel:C1234567890"
    ```
  </Tab>
  <Tab title="Overschrijving van model en denken">
    ```bash
    openclaw cron add \
      --name "Deep analysis" \
      --cron "0 6 * * 1" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --message "Weekly deep analysis of project progress." \
      --model "opus" \
      --thinking high \
      --announce
    ```
  </Tab>
  <Tab title="Webhookuitvoer">
    ```bash
    openclaw cron create "0 18 * * 1-5" \
      "Summarize today's deploys as JSON." \
      --name "Deploy digest" \
      --webhook "https://example.invalid/openclaw/cron"
    ```
  </Tab>
  <Tab title="Opdrachtuitvoer">
    ```bash
    openclaw cron create "*/15 * * * *" \
      --name "Queue depth probe" \
      --command "scripts/check-queue.sh" \
      --command-cwd "/srv/app" \
      --announce \
      --channel telegram \
      --to "-1001234567890"
    ```
  </Tab>
</Tabs>

## Taken beheren

```bash
# Ingeschakelde taken weergeven
openclaw cron list

# Uitgeschakelde taken opnemen
openclaw cron list --all

# Eén opgeslagen taak als JSON ophalen
openclaw cron get <jobId>

# Eén taak weergeven, inclusief de herleide afleveringsroute
openclaw cron show <jobId>

# In-/uitschakelen zonder te verwijderen
openclaw cron enable <jobId>
openclaw cron disable <jobId>

# Een taak bewerken
openclaw cron edit <jobId> --message "Bijgewerkte prompt" --model "opus"

# Een taak nu geforceerd uitvoeren
openclaw cron run <jobId>

# Een taak nu geforceerd uitvoeren en op de eindstatus wachten
openclaw cron run <jobId> --wait --wait-timeout 10m --poll-interval 2s

# Alleen uitvoeren als de taak aan de beurt is
openclaw cron run <jobId> --due

# Uitvoeringsgeschiedenis bekijken
openclaw cron runs --id <jobId> --limit 50

# Eén specifieke uitvoering bekijken
openclaw cron runs --id <jobId> --run-id <runId>

# Een taak verwijderen
openclaw cron remove <jobId>

# Agentselectie (opstellingen met meerdere agents)
openclaw cron create "0 6 * * *" "Controleer de operationele wachtrij" --name "Operationele controle" --session isolated --agent ops
openclaw cron edit <jobId> --clear-agent
```

Als je een sessie archiveert (via de Control UI of `sessions.patch { archived: true }` vanuit een aanroepende operatorbeheerder), worden alle ingeschakelde cron-taken uitgeschakeld die aan die sessie zijn gekoppeld: de geïsoleerde `cron:<jobId>`-sessie ervan, een `session:<key>`-doel of een `sessionKey`-route voor aflevering/activering. Door de sessie te herstellen, worden deze taken niet opnieuw ingeschakeld; gebruik `openclaw cron enable <jobId>`. Sessies met een ingeschakelde gekoppelde taak tonen een klokbadge in de zijbalk van de Control UI.

`openclaw cron run <jobId>` keert terug nadat de handmatige uitvoering in de wachtrij is geplaatst. Gebruik `--wait` voor afsluitingshooks, onderhoudsscripts of andere automatisering die moet blokkeren totdat de uitvoering in de wachtrij is voltooid; dit peilt de geretourneerde `runId` (standaardtime-out `10m`, peilinterval `2s`) en sluit af met `0` voor status `ok`, en met een andere waarde dan nul voor `error`, `skipped` of een time-out tijdens het wachten.

De agenttool `cron` retourneert compacte taaksamenvattingen (`id`, `name`, `enabled`, `nextRunAtMs`, `scheduleKind`, `lastRunStatus`) vanuit `cron(action: "list")`; gebruik `cron(action: "get", jobId: "...")` voor één volledige taakdefinitie. Rechtstreekse Gateway-aanroepers kunnen `compact: true` doorgeven aan `cron.list`; als je dit weglaat, blijft de volledige respons met afleveringsvoorbeelden behouden.

`openclaw cron create` is een alias voor `openclaw cron add`. Nieuwe taken kunnen een positioneel schema gebruiken (`"0 9 * * 1"`, `"every 1h"`, `"20m"` of een ISO-tijdstempel), gevolgd door een positionele agentprompt. Gebruik `--webhook <url>` bij `cron add|create` of `cron edit` om de voltooide uitvoeringspayload via POST naar een HTTP-eindpunt te sturen; Webhook-aflevering kan niet worden gecombineerd met chat-afleveringsvlaggen (`--announce`, `--channel`, `--to`, `--thread-id`, `--account`). Bij `cron edit`, `--clear-channel`, `--clear-to`, `--clear-thread-id` en `--clear-account` worden deze routeringsvelden afzonderlijk verwijderd (elk wordt geweigerd naast de bijbehorende instelvlag) — anders dan `--no-deliver`, dat alleen de terugvalaflevering van de uitvoerder uitschakelt.

<Note>
Opmerking over modeloverschrijving:

- `openclaw cron add|edit --model ...` wijzigt het geselecteerde model van de taak.
- Als het model is toegestaan, bereikt die exacte provider/dat model de geïsoleerde agentuitvoering.
- Als het niet is toegestaan of niet kan worden herleid, laat cron de uitvoering mislukken met een expliciete validatiefout.
- Payloadpatches voor API `cron.update` kunnen `model: null` instellen om een opgeslagen modeloverschrijving voor een taak te wissen.
- `openclaw cron edit <job-id> --clear-model` wist die overschrijving vanuit de CLI (hetzelfde effect als de patch `model: null`) en kan niet worden gecombineerd met `--model`.
- Geconfigureerde terugvalketens blijven van toepassing omdat cron `--model` een primaire taakinstelling is en geen overschrijving van sessie-`/model`.
- `openclaw cron add|edit --fallbacks ...` stelt payload `fallbacks` in en vervangt de geconfigureerde terugvalopties voor die taak; `--fallbacks ""` schakelt terugval uit en maakt de uitvoering strikt. `openclaw cron edit <job-id> --clear-fallbacks` wist de overschrijving per taak.
- Een gewone `--model` zonder expliciete of geconfigureerde terugvallijst valt niet terug op de primaire agent als stil extra doel voor een nieuwe poging.

</Note>

## Webhooks

Gateway kan HTTP-Webhook-eindpunten beschikbaar stellen voor externe triggers. Schakel dit in de configuratie in:

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### Authenticatie

Elk verzoek moet het hooktoken via een header bevatten:

- `Authorization: Bearer <token>` (aanbevolen)
- `x-openclaw-token: <token>`

Tokens in de querystring worden geweigerd.

<AccordionGroup>
  <Accordion title="POST /hooks/wake">
    Plaats een systeemgebeurtenis in de wachtrij voor de hoofdsessie:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/wake \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"text":"Nieuwe e-mail ontvangen","mode":"now"}'
    ```

    <ParamField path="text" type="string" required>
      Beschrijving van de gebeurtenis.
    </ParamField>
    <ParamField path="mode" type="string" default="now">
      `now` of `next-heartbeat`.
    </ParamField>

  </Accordion>
  <Accordion title="POST /hooks/agent">
    Voer een geïsoleerde agentbeurt uit:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/agent \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"message":"Vat de inbox samen","name":"E-mail","model":"openai/gpt-5.6-sol"}'
    ```

    Velden: `message` (verplicht), `name`, `agentId`, `sessionKey` (vereist `hooks.allowRequestSessionKey=true`), `idempotencyKey`, `wakeMode`, `deliver`, `channel`, `to`, `model`, `thinking`, `timeoutSeconds`.

  </Accordion>
  <Accordion title="Toegewezen hooks (POST /hooks/<name>)">
    Aangepaste hooknamen worden via `hooks.mappings` in de configuratie herleid. Toewijzingen kunnen willekeurige payloads met sjablonen of codetransformaties omzetten in acties voor `wake` of `agent`.
  </Accordion>
</AccordionGroup>

<Warning>
Houd hookeindpunten achter loopback, tailnet of een vertrouwde reverse proxy.

- Gebruik een speciaal hooktoken; hergebruik geen authenticatietokens van Gateway.
- Houd `hooks.path` op een afzonderlijk subpad; `/` wordt geweigerd.
- Stel `hooks.allowedAgentIds` in om te beperken op welke effectieve agent een hook zich kan richten, inclusief de standaardagent wanneer `agentId` wordt weggelaten.
- Behoud `hooks.allowRequestSessionKey=false`, tenzij je door de aanroeper geselecteerde sessies nodig hebt.
- Als je `hooks.allowRequestSessionKey` inschakelt, stel dan ook `hooks.allowedSessionKeyPrefixes` in om toegestane vormen van sessiesleutels te beperken.
- Hookpayloads worden standaard met veiligheidsgrenzen omgeven.

</Warning>

## Gmail PubSub-integratie

Verbind triggers voor het Gmail-postvak met OpenClaw via Google PubSub.

<Note>
**Vereisten:** `gcloud` CLI, `gog` (gogcli), ingeschakelde OpenClaw-hooks en Tailscale voor het openbare HTTPS-eindpunt.
</Note>

### Installatie via wizard (aanbevolen)

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

Dit schrijft de configuratie `hooks.gmail`, schakelt de Gmail-voorinstelling in en gebruikt standaard Tailscale Funnel voor het push-eindpunt (`--tailscale funnel|serve|off`).

<Warning>
De sessie per bericht van de Gmail-voorinstelling scheidt de gesprekscontext; deze beperkt niet de tools of werkruimte van de doelagent. Zonder een aangepaste toewijzing die `agentId` instelt, worden Gmail-hooks als de standaardagent uitgevoerd.

Routeer voor niet-vertrouwde inboxen de hook naar een speciale leesagent, geef die agent alleen-lezen- of geen toegang tot de werkruimte en weiger schrijftoegang tot het bestandssysteem, shell, browser en andere onnodige tools. Als deze de hoofdagent moet informeren, sta dan alleen de vereiste overdracht tussen agents toe. Zie [Promptinjectie](/nl/gateway/security#prompt-injection), [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) en [`tools.agentToAgent`](/nl/gateway/config-tools#toolsagenttoagent).
</Warning>

### Gateway automatisch starten

Wanneer `hooks.enabled=true` en `hooks.gmail.account` zijn ingesteld, start Gateway `gog gmail watch serve` tijdens het opstarten en vernieuwt het de bewaking automatisch. Stel `OPENCLAW_SKIP_GMAIL_WATCHER=1` in om dit uit te schakelen.

### Eenmalige handmatige installatie

<Steps>
  <Step title="Selecteer het GCP-project">
    Selecteer het GCP-project dat eigenaar is van de OAuth-client die door `gog` wordt gebruikt:

    ```bash
    gcloud auth login
    gcloud config set project <project-id>
    gcloud services enable gmail.googleapis.com pubsub.googleapis.com
    ```

  </Step>
  <Step title="Maak het onderwerp en verleen Gmail-pushtoegang">
    ```bash
    gcloud pubsub topics create gog-gmail-watch
    gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
      --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
      --role=roles/pubsub.publisher
    ```
  </Step>
  <Step title="Start de bewaking">
    ```bash
    gog gmail watch start \
      --account openclaw@gmail.com \
      --label INBOX \
      --topic projects/<project-id>/topics/gog-gmail-watch
    ```
  </Step>
</Steps>

### Gmail-modeloverschrijving

```json5
{
  hooks: {
    gmail: {
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

Gebruik voor niet-vertrouwde inboxen het beste beschikbare model van de nieuwste generatie van je provider. De bovenstaande waarde is een voorbeeld; het model moet in je geconfigureerde catalogus en toelatingslijst voorkomen.

## Configuratie

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    triggers: {
      enabled: false,
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
  },
}
```

`webhookToken` wordt als `Authorization: Bearer <token>` verzonden bij cron-Webhook-POST-verzoeken.

`cron.store` is een logische opslagsleutel en migratiepad voor doctor, geen actief JSON-bestand om handmatig te bewerken. Taakgegevens bevinden zich in SQLite; gebruik de CLI of Gateway-API voor wijzigingen.

Schakel cron uit: `cron.enabled: false` of `OPENCLAW_SKIP_CRON=1`.

<AccordionGroup>
  <Accordion title="Gedrag bij nieuwe pogingen">
    **Nieuwe poging voor eenmalige taken**: tijdelijke fouten (snelheidslimiet, overbelasting, netwerk, time-out, serverfout) gebruiken een ingebouwd schema voor nieuwe pogingen. Permanente fouten schakelen de taak onmiddellijk uit.

    **Nieuwe poging voor terugkerende taken**: bij opeenvolgende uitvoeringsfouten wordt volgens een uitgebreid schema langer gewacht (30s, 60s, 5m, 15m, 60m). Na de volgende geslaagde uitvoering wordt de wachttijd opnieuw ingesteld.

  </Accordion>
  <Accordion title="Onderhoud">
    `cron.sessionRetention` (standaard `24h`; `false` schakelt dit uit) ruimt geïsoleerde vermeldingen van uitvoeringssessies op. De uitvoeringsgeschiedenis bewaart per taak de nieuwste 2000 eindstatusrijen; verloren rijen behouden hun opschoonvenster van 24 uur.
  </Accordion>
  <Accordion title="Migratie van verouderde opslag">
    Voer na een upgrade `openclaw doctor --fix` uit om verouderde bestanden `~/.openclaw/cron/jobs.json`, `jobs-state.json` en `runs/*.jsonl` in SQLite te importeren en ze te hernoemen met het achtervoegsel `.migrated`. Ongeldige taakrijen worden tijdens runtime overgeslagen en voor latere reparatie of controle naar `jobs-quarantine.json` gekopieerd.
  </Accordion>
</AccordionGroup>

## Problemen oplossen

### Opdrachtvolgorde

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

<AccordionGroup>
  <Accordion title="Cron wordt niet geactiveerd">
    - Controleer `cron.enabled` en de omgevingsvariabele `OPENCLAW_SKIP_CRON`.
    - Controleer of Gateway continu actief is.
    - Controleer bij `cron`-schema's de tijdzone (`--tz`) ten opzichte van de tijdzone van de host.
    - `reason: not-due` in de uitvoer betekent dat de handmatige uitvoering met `openclaw cron run <jobId> --due` is gecontroleerd en dat de taak nog niet aan de beurt was.

  </Accordion>
  <Accordion title="Cron is geactiveerd, maar er is niets afgeleverd">
    - Afleveringsmodus `none` betekent dat er geen terugvalverzending door de runner wordt verwacht. De agent kan nog steeds rechtstreeks verzenden met de tool `message` wanneer er een chatroute beschikbaar is.
    - Een ontbrekend/ongeldig afleveringsdoel (`channel`/`to`) betekent dat uitgaande verzending is overgeslagen.
    - Voor Matrix kunnen gekopieerde of verouderde taken met kamer-ID's in kleine letters voor `delivery.to` mislukken, omdat Matrix-kamer-ID's hoofdlettergevoelig zijn. Bewerk de taak met de exacte waarde `!room:server` of `room:!room:server` uit Matrix.
    - Authenticatiefouten van het kanaal (`unauthorized`, `Forbidden`) betekenen dat aflevering door de inloggegevens is geblokkeerd.
    - Als de geïsoleerde uitvoering alleen het stille token (`NO_REPLY` / `no_reply`) retourneert, onderdrukt OpenClaw rechtstreekse uitgaande aflevering en het terugvalpad met een samenvatting in de wachtrij, zodat er niets naar de chat wordt teruggestuurd.
    - Als de agent de gebruiker zelf een bericht moet sturen, controleer dan of de taak een bruikbare route heeft (`channel: "last"` met een eerdere chat, of een expliciet kanaal/doel).

  </Accordion>
  <Accordion title="Cron of Heartbeat lijkt een rollover in /new-stijl te voorkomen">
    - De actualiteit voor dagelijkse en inactiviteitsresets is niet gebaseerd op `updatedAt`; zie [Sessiebeheer](/nl/concepts/session#session-lifecycle).
    - Cron-activeringen, Heartbeat-uitvoeringen, uitvoeringsmeldingen en Gateway-administratie kunnen de sessierij bijwerken voor routering/status, maar verlengen `sessionStartedAt` of `lastInteractionAt` niet.
    - Voor verouderde rijen die zijn gemaakt voordat die velden bestonden, kan OpenClaw `sessionStartedAt` herstellen uit de JSONL-sessieheader van het transcript wanneer het bestand nog beschikbaar is. Verouderde inactieve rijen zonder `lastInteractionAt` gebruiken die herstelde starttijd als uitgangspunt voor hun inactiviteit.

  </Accordion>
  <Accordion title="Valkuilen met tijdzones">
    - Cron zonder `--tz` gebruikt de tijdzone van de Gateway-host.
    - `at`-schema's zonder tijdzone worden als UTC behandeld.
    - Heartbeat `activeHours` gebruikt de geconfigureerde tijdzonebepaling.

  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Automatisering](/nl/automation) — alle automatiseringsmechanismen in één oogopslag
- [Achtergrondtaken](/nl/automation/tasks) — taaklogboek voor Cron-uitvoeringen
- [Heartbeat](/nl/gateway/heartbeat) — periodieke beurten in de hoofdsessie
- [Tijdzone](/nl/concepts/timezone) — tijdzoneconfiguratie
